# Методическое пособие по интеграции поиска карточек по изображению.

В данном методическом пособии на примере магазина мебели будет показано применение <a href="https://huggingface.co/docs/transformers.js/index">`transformers.js`</a> для полной реализации поиска карточки с подходящим описанием по картинке через CLIP-модель. 

Для начала мы реализуем простое React-приложение интернет-магазина мебели с простым текстовым происком. Затем мы уже интегрируем поиск по изображению.
Первичное приложение будет во многом повторять ЛР по React, так что задерживаться на создании сильно не будем. 

## Создание простого React-приложения

### 1. Инициализация проекта

Откройте терминал и выполните следующую команду для создания проекта с использованием Vite:

```bash
npm create vite@latest furniture-shop -- --template react-ts
```

Перейдите в папку проекта:

```bash
cd furniture-shop
```

Установите базовые зависимости:

```bash
npm install
```

Установите дополнительные библиотеки (маршрутизация, стили, ML-библиотеки):

```bash
npm install react-router-dom bootstrap react-bootstrap @huggingface/transformers 
```

### 2. Структура проекта

Создайте следующую структуру папок внутри `src`:

```
src/
├── assets/          # Изображения (поместите сюда файлы 1.jpg, 2.jpg и т.д.)
├── components/      # Переиспользуемые компоненты
├── modules/         # Типы и моковые данные
├── pages/           # Страницы приложения
├── workers/         # Web Workers (для ML задач)
├── App.css
├── App.tsx
├── index.css
├── main.tsx
└── vite-env.d.ts
```

### 3. Настройка конфигурации

#### `vite.config.ts`
Настроим порт сервера разработки на 3000.

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vitejs.dev/config/
export default defineConfig({
  server: { port: 3000 },
  plugins: [react()],
})
```

### 4. Базовые файлы приложения

#### `src/main.tsx`
Точка входа в приложение. Подключаем стили Bootstrap.

```tsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.tsx'
import './index.css'
import 'bootstrap/dist/css/bootstrap.min.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```



#### `src/App.tsx`
Настройка маршрутизации.

```tsx
import { BrowserRouter, Route, Routes, Navigate } from "react-router-dom";
import { HomePage } from "./pages/HomePage";
import { CatalogPage } from "./pages/CatalogPage";
import { ProductPage } from "./pages/ProductPage";
import 'bootstrap/dist/css/bootstrap.min.css';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        {/* Главная */}
        <Route path="/" element={<HomePage />} />
        
        {/* Каталог */}
        <Route path="/catalog" element={<CatalogPage />} />
        
        {/* Товар */}
        <Route path="/catalog/:id" element={<ProductPage />} />

        {/* Если путь не найден, редирект на главную */}
        <Route path="*" element={<Navigate to="/" replace />} />
      </Routes>
    </BrowserRouter>
  );
}

export default App;
```

### 5. Данные

#### `src/modules/mock.ts`
Определение типов и моковых данных. Убедитесь, что в папке `src/assets` есть изображения `1.jpg` ... `8.jpg`. Взять их можно в ...

```typescript
import img1 from '../assets/1.jpg';
import img2 from '../assets/2.jpg';
import img3 from '../assets/3.jpg';
import img4 from '../assets/4.jpg';
import img5 from '../assets/5.jpg';
import img6 from '../assets/6.jpg';
import img7 from '../assets/7.jpg';
import img8 from '../assets/8.jpg';

export interface IFurniture {
    id: number;
    name: string;
    description: string;
    price: number;
    image: string;
}

export const FURNITURE_MOCK: IFurniture[] = [
    {
        id: 1,
        name: "Диван 'Облако'",
        description: "Мягкий белый трехместный диван с качественной обивкой. Идеально подходит для гостиной в скандинавском стиле.",
        price: 45990,
        image: img1
    },
    {
        id: 2,
        name: "Кресло 'Ретро'",
        description: "Удобное кресло на деревянных ножках. Классический дизайн 60-х годов в современном исполнении.",
        price: 12500,
        image: img2
    },
    {
        id: 3,
        name: "Стол обеденный 'Дуб'",
        description: "Массивный стол из натурального дуба. Покрыт защитным маслом. Вмещает до 6 человек.",
        price: 28000,
        image: img3
    },
    {
        id: 4,
        name: "Стул 'Эймс'",
        description: "Эргономичный пластиковый стул с деревянными ножками. Легкий, прочный и стильный.",
        price: 3500,
        image: img4
    },
    {
        id: 5,
        name: "Торшер 'Лофт'",
        description: "Стильный металлический торшер в индустриальном стиле. Регулируемая высота и наклон плафона.",
        price: 4200,
        image: img5
    },
    {
        id: 6,
        name: "Комод 'Винтаж'",
        description: "Вместительный комод с тремя ящиками. Искусственно состаренная поверхность придает особый шарм.",
        price: 18900,
        image: img6
    },
    {
        id: 7,
        name: "Полка навесная",
        description: "Лаконичная полка для книг и декора. Скрытое крепление создает эффект парения.",
        price: 1500,
        image: img7
    },
    {
        id: 8,
        name: "Пуф вязаный",
        description: "Уютный вязаный пуф ручной работы. Отлично дополнит зону отдыха.",
        price: 2900,
        image: img8
    }
];
```

### 6. Компоненты

#### `src/components/BreadCrumbs.tsx`
Компонент "хлебные крошки" для навигации.

```tsx
import "./BreadCrumbs.css";
import React from "react";
import { Link } from "react-router-dom";
import type { FC } from "react";

interface ICrumb {
  label: string;
  path?: string;
}

interface BreadCrumbsProps {
  crumbs: ICrumb[];
}

export const BreadCrumbs: FC<BreadCrumbsProps> = (props) => {
  const { crumbs } = props;

  return (
    <ul className="breadcrumbs">
      <li>
        <Link to="/">Главная</Link>
      </li>
      {!!crumbs.length &&
        crumbs.map((crumb, index) => (
          <React.Fragment key={index}>
            <li className="slash">/</li>
            {index === crumbs.length - 1 ? (
              <li>{crumb.label}</li>
            ) : (
              <li>
                <Link to={crumb.path || ""}>{crumb.label}</Link>
              </li>
            )}
          </React.Fragment>
        ))}
    </ul>
  );
};
```

#### `src/components/BreadCrumbs.css`
```css
:root {
  --active_color: black;
  --additional_color: gray;
}
.breadcrumbs {
  list-style: none;
  display: flex;
  gap: 10px;
  padding: 20px 0; 
}
.breadcrumbs * {
  color: var(--additional_color);
  text-decoration: none;
  transition: 0.3s;
}
.breadcrumbs *:not(.slash):hover {
  color: var(--active_color);
}
.breadcrumbs li:last-child {
  color: var(--active_color);
  font-weight: bold;
  cursor: default;
}
```

#### `src/components/FurnitureCard.tsx`
Карточка товара.

```tsx
import type { FC } from "react";
import { Button, Card } from "react-bootstrap";
import "./FurnitureCard.css";

interface Props {
  id: number;
  name: string;
  description: string;
  price: number;
  image: string;
  onClick: (id: number) => void;
}

export const FurnitureCard: FC<Props> = ({ id, name, description, price, image, onClick }) => {
  return (
    <Card className="furniture-card h-100 shadow-sm">
      <div className="img-wrapper" onClick={() => onClick(id)}>
        <Card.Img variant="top" src={image} className="furniture-img" />
      </div>
      <Card.Body className="d-flex flex-column">
        <Card.Title>{name}</Card.Title>
        <Card.Text className="description text-muted">
          {description}
        </Card.Text>
        <div className="mt-auto">
          <h5 className="price-tag">{price.toLocaleString('ru-RU')} ₽</h5>
          <Button variant="dark" className="w-100" onClick={() => onClick(id)}>
            Подробнее
          </Button>
        </div>
      </Card.Body>
    </Card>
  );
};
```

#### `src/components/FurnitureCard.css`
```css
.furniture-card {
    transition: transform 0.2s;
    cursor: pointer;
}
.furniture-card:hover {
    transform: translateY(-5px);
}
.img-wrapper {
    height: 200px;
    overflow: hidden;
}
.furniture-img {
    height: 100%;
    width: 100%;
    object-fit: cover; 
}
.description {
    display: -webkit-box;
    -webkit-line-clamp: 3;
    -webkit-box-orient: vertical;
    overflow: hidden;
    font-size: 0.9rem;
}
.price-tag {
    font-weight: bold;
    color: #333;
    margin-bottom: 15px;
}
```

#### `src/components/SearchBar.tsx`
Поле поиска.

```tsx
import type { FC } from 'react';
import { Button, Form, InputGroup } from 'react-bootstrap';

interface Props {
    value: string;
    setValue: (value: string) => void;
}

export const SearchBar: FC<Props> = ({ value, setValue }) => (
    <InputGroup className="mb-4">
        <Form.Control
            placeholder="Что вы ищете? (например, диван)"
            value={value}
            onChange={(e) => setValue(e.target.value)}
        />
        <Button variant="outline-secondary" onClick={() => setValue('')}>
            Очистить
        </Button>
    </InputGroup>
)
```

### 7. Страницы

#### `src/pages/HomePage.tsx`
Главная страница.

```tsx
import type { FC } from "react";
import { Link } from "react-router-dom";
import { Button, Container } from "react-bootstrap";

export const HomePage: FC = () => {
  return (
    <div style={{
        minHeight: "100vh", 
        width: "100%",
        display: "flex", 
        alignItems: "center", 
        justifyContent: "center",
        flexDirection: "column",
        backgroundColor: "#f8f9fa",
        padding: "20px"
    }}>
      <Container className="text-center">
        <h1 className="display-1 fw-bold mb-3">Modern Living</h1>
        <p className="lead mb-5 fs-3">
          Создайте уют с нашей новой коллекцией мебели
        </p>
        <Link to="/catalog">
          <Button variant="dark" size="lg" className="px-5 py-3 fs-5">
            Перейти в каталог
          </Button>
        </Link>
      </Container>
    </div>
  );
};
```

#### `src/pages/CatalogPage.tsx`
Страница каталога с поиском и списком товаров.

```tsx
import { useState, useEffect } from "react";
import type { FC } from "react";
import { Container, Row, Col } from "react-bootstrap";
import { useNavigate } from "react-router-dom";
import { FURNITURE_MOCK } from "../modules/mock";
import type { IFurniture } from "../modules/mock";
import { FurnitureCard } from "../components/FurnitureCard";
import { SearchBar } from "../components/SearchBar";
import { BreadCrumbs } from "../components/BreadCrumbs";

export const CatalogPage: FC = () => {
  const [searchTerm, setSearchTerm] = useState("");
  const [displayedItems, setDisplayedItems] = useState<IFurniture[]>(FURNITURE_MOCK);
  
  const navigate = useNavigate();

  useEffect(() => {
    const filtered = FURNITURE_MOCK.filter(item => 
        item.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
        item.description.toLowerCase().includes(searchTerm.toLowerCase())
    );
    setDisplayedItems(filtered);
  }, [searchTerm]);

  const handleCardClick = (id: number) => {
    navigate(`/catalog/${id}`);
  };

  return (
    <Container className="py-4">
      <BreadCrumbs crumbs={[{ label: "Каталог" }]} />
      
      <h1 className="mb-4">Каталог мебели</h1>
      
      <SearchBar 
        value={searchTerm} 
        setValue={setSearchTerm} 
      />

      {displayedItems.length === 0 ? (
         <div className="text-center mt-5">
            <h3>Ничего не найдено :(</h3>
            <p>Попробуйте изменить запрос</p>
         </div>
      ) : (
        <Row xs={1} md={2} lg={4} className="g-4">
            {displayedItems.map((item) => (
                <Col key={item.id}>
                    <FurnitureCard 
                        {...item} 
                        onClick={handleCardClick}
                    />
                </Col>
            ))}
        </Row>
      )}
    </Container>
  );
};
```

#### `src/pages/ProductPage.tsx`
Страница отдельного товара.

```tsx
import type { FC } from "react";
import { useParams } from "react-router-dom";
import { Container, Row, Col, Image, Button, Badge } from "react-bootstrap";
import { BreadCrumbs } from "../components/BreadCrumbs";
import { FURNITURE_MOCK } from "../modules/mock";

export const ProductPage: FC = () => {
  const { id } = useParams();
  
  const product = FURNITURE_MOCK.find(item => String(item.id) === id);

  if (!product) {
      return (
          <Container className="py-5 text-center">
              <h2>Товар не найден</h2>
              <Button href="/catalog" variant="primary">Вернуться в каталог</Button>
          </Container>
      )
  }

  return (
    <Container className="py-4">
      <BreadCrumbs
        crumbs={[
          { label: "Каталог мебели", path: "/catalog" },
          { label: product.name },
        ]}
      />
      
      <Row className="mt-4">
        <Col md={6} className="mb-4">
          <Image src={product.image} alt={product.name} fluid rounded className="shadow" />
        </Col>
        
        <Col md={6}>
          <Badge bg="secondary" className="mb-2">ID: {product.id}</Badge>
          <h1 className="display-5">{product.name}</h1>
          <h2 className="text-primary my-3">{product.price.toLocaleString('ru-RU')} ₽</h2>
          <p className="lead">{product.description}</p>
          
          <div className="d-grid gap-2 mt-5">
            <Button variant="dark" size="lg">Добавить в корзину</Button>
            <Button variant="outline-dark">Купить в 1 клик</Button>
          </div>
        </Col>
      </Row>
    </Container>
  );
};
```

### 8. Глобальные стили

#### `src/index.css`
```css
:root {
  font-family: system-ui, Avenir, Helvetica, Arial, sans-serif;
  line-height: 1.5;
  font-weight: 400;
  color-scheme: light dark;
  color: rgba(255, 255, 255, 0.87);
  background-color: #242424;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}
a:hover {
  color: #535bf2;
}

body {
  margin: 0;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Fira Sans', 'Droid Sans', 'Helvetica Neue',
    sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  min-height: 100vh;
}

h1 {
  font-size: 3.2em;
  line-height: 1.1;
}

button {
  border-radius: 8px;
  border: 1px solid transparent;
  padding: 0.6em 1.2em;
  font-size: 1em;
  font-weight: 500;
  font-family: inherit;
  background-color: #1a1a1a;
  cursor: pointer;
  transition: border-color 0.25s;
}
```

### Запуск проекта

```bash
npm run dev
```

Тут у вас должно получиться простое React-приложение интернет магазина.

(фотографии)


## Интеграция CLIP-модели

### Теория про CLIP

Для реализации поиска по изображению в данной работе используются технологии глубокого обучения (Deep Learning), основанные на архитектуре Трансформер (Transformer). Чтобы понять, как именно происходит "магия" сопоставления картинки и текста, разберем ключевые компоненты системы.

#### 1. Трансформер как «черный ящик»
Архитектура Трансформер, совершившая революцию в обработке естественного языка (NLP), состоит из двух основных блоков:
*   **Энкодер (Encoder)** - отвечает за понимание входных данных и перевод их в векторное представление.
*   **Декодер (Decoder)** - отвечает за генерацию новых данных на основе информации от энкодера.

В контексте нашей задачи нас интересует только **Энкодер**. Можно представить его как «черный ящик»: на вход он получает данные (например, предложение), а на выходе выдает **эмбеддинг** - плотный числовой вектор фиксированной длины. Этот вектор является "смысловым слепком" входных данных: если два предложения похожи по смыслу, их векторы-эмбеддинги будут находиться близко друг к другу в математическом пространстве.

#### 2. Визуальные трансформеры (Vision Transformers)
Долгое время трансформеры применялись только к тексту. Однако выяснилось, что Энкодер трансформера - устройство универсальное. Ему неважно, что именно подается на вход, главное - представить данные как последовательность элементов.

Чтобы скормить Энкодеру изображение, применяется следующий алгоритм (Vision Transformer, ViT):
1.  **Нарезка на патчи:** Изображение разбивается на сетку квадратов (например, 16x16 пикселей).
2.  **Линеаризация:** Каждый квадрат (патч) "вытягивается" в плоскую последовательность пикселей. Теперь картинка для модели выглядит как набор фрагментов, аналогично тому, как предложение выглядит как набор слов.
3.  **Позиционное кодирование (Positional Encoding):** Чтобы модель понимала, где находится какой фрагмент (где левый верхний угол, а где центр), к данным каждого патча добавляется специальный вектор позиции.
4.  **Эмбеддинг:** Подготовленная последовательность проходит через слои Энкодера, и на выходе мы получаем один итоговый вектор - **эмбеддинг изображения**.

#### 3. Модель CLIP: единое смысловое пространство
Проблема классических подходов в том, что модели для текста и модели для картинок существуют в разных "мирах". Вектор слова "собака" и вектор фотографии собаки, полученные разными нейросетями, математически никак не связаны. Их нельзя просто взять и сравнить.

Здесь на сцену выходит **CLIP (Contrastive Language-Image Pre-training)**.
CLIP - это мультимодальная архитектура, которая обучает два Энкодера одновременно:
1.  **Text Encoder:** превращает текст в вектор.
2.  **Image Encoder:** превращает картинку в вектор.

Главная особенность CLIP заключается в том, что эти два энкодера обучаются так, чтобы **проецировать данные в общее смысловое (векторное) пространство**. В процессе обучения модель видит миллионы пар "картинка + описание" и настраивает свои веса так, чтобы:
*   Вектор изображения собаки и вектор текста "фотография собаки" были **близки**.
*   Вектор изображения собаки и вектор текста "тарелка супа" были **далеки** друг от друга.

#### 4. Применение для поиска
Благодаря тому, что CLIP (и его современные вариации, такие как SigLIP) создает единое пространство для разных модальностей, мы можем реализовать поиск без использования сложной классификации и разметки данных:
1.  Берем изображение от пользователя и прогоняем через **Image Encoder** $\rightarrow$ получаем вектор $V_{img}$.
2.  Берем описания всех товаров в базе и прогоняем через **Text Encoder** $\rightarrow$ получаем набор векторов $V_{text\_1}, V_{text\_2}, \dots$.
3.  Считаем **косинусное сходство** (Cosine Similarity) между $V_{img}$ и каждым из текстовых векторов.
4.  Товары, чьи текстовые векторы оказались "ближе" всего к вектору картинки, и являются искомыми объектами.

В данной лабораторной работе мы используем модель <a href="https://huggingface.co/Xenova/siglip-base-patch16-224">SigLIP</a> (улучшенную версию CLIP от Google), которая работает по тому же принципу, но обладает более высокой точностью и поддержкой множества языков.

### Что такое transfomers.js?

Простыми словами, `transformers.js` - это перевод библиотеки <a href="https://huggingface.co/docs/transformers/index">HF Transformers</a> с python на JavaScript. Сущности из оригинальной библиотеки постарались перенести в JS, также некоторые модели, которые раньше запускались **только** через python, теперь могут работать на JavaScript. 

Теперь быстренько перепишем интерфейс под новые функции, и подронбно разберем саму логику работы с `transfomers.js` в нашем проекте. 


### `src/modules/math.ts`
Создадим вспомогательный файл для математических вычислений. Здесь находится функция для расчета косинусного сходства векторов.

```typescript
export function cosineSimilarity(vecA: number[], vecB: number[]): number {
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;

    for (let i = 0; i < vecA.length; i++) {
        dotProduct += vecA[i] * vecB[i];
        normA += vecA[i] * vecA[i];
        normB += vecB[i] * vecB[i];
    }

    if (normA === 0 || normB === 0) return 0;
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

### `src/components/SearchBar.tsx`
Обновим компонент поиска. Теперь он умеет принимать изображения через скрытый input file.

```tsx
import { useRef } from 'react';
import type {FC} from 'react';
import { Button, Form, InputGroup, Spinner } from 'react-bootstrap';

interface Props {
    value: string;
    setValue: (value: string) => void;
    onImageSelect: (file: File) => void;
    isProcessing?: boolean;
}

export const SearchBar: FC<Props> = ({ value, setValue, onImageSelect, isProcessing }) => {
    const fileInputRef = useRef<HTMLInputElement>(null);

    const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
        if (e.target.files && e.target.files[0]) {
            onImageSelect(e.target.files[0]);
        }
    };

    return (
        <InputGroup className="mb-4">
            <Form.Control
                placeholder="Что вы ищете? (текст или фото)"
                value={value}
                onChange={(e) => setValue(e.target.value)}
                disabled={isProcessing}
            />
            {/* Скрытый инпут для файла */}
            <input 
                type="file" 
                accept="image/*" 
                ref={fileInputRef} 
                style={{ display: 'none' }} 
                onChange={handleFileChange}
            />
            
            <Button 
                variant="outline-secondary" 
                onClick={() => fileInputRef.current?.click()}
                disabled={isProcessing}
                title="Поиск по картинке"
            >
                {isProcessing ? <Spinner animation="border" size="sm" /> : "📷 Фото"}
            </Button>

            <Button 
                variant="outline-secondary" 
                onClick={() => setValue('')}
                disabled={isProcessing}
            >
                Очистить
            </Button>
        </InputGroup>
    );
}
```

### `CatalogPage.tsx`

Добавим загрузку отображение загрузки моделей в каталок.

```tsx
import { useState, useEffect, useRef } from "react";
import type { FC } from "react";
import { Container, Row, Col, ProgressBar } from "react-bootstrap";
import { useNavigate } from "react-router-dom";
import { FURNITURE_MOCK } from "../modules/mock";
import type { IFurniture } from "../modules/mock";
import { FurnitureCard } from "../components/FurnitureCard";
import { SearchBar } from "../components/SearchBar";
import { BreadCrumbs } from "../components/BreadCrumbs";

export const CatalogPage: FC = () => {
  const [searchTerm, setSearchTerm] = useState("");
  const [displayedItems, setDisplayedItems] = useState<IFurniture[]>(FURNITURE_MOCK);
  
  // Состояния для ML поиска
  const [isProcessing, setIsProcessing] = useState(false);
  const [progress, setProgress] = useState(0);
  const [loadingText, setLoadingText] = useState("");
  const [searchMode, setSearchMode] = useState<'text' | 'image'>('text');

  const navigate = useNavigate();
  const workerRef = useRef<Worker | null>(null);

  // Инициализация воркера при монтировании компонента
  useEffect(() => {
    workerRef.current = new Worker(new URL('../workers/search.worker.ts', import.meta.url), {
        type: 'module'
    });

    workerRef.current.onmessage = (event) => {
        const { status, data, results } = event.data;

        if (status === 'progress') {
            // data.status может быть 'initiate', 'download', 'progress', 'done'
            if (data.status === 'progress') {
                setProgress(data.progress);
                setLoadingText(`Загрузка модели: ${Math.round(data.progress)}%`);
            } else if (data.status === 'initiate') {
                setLoadingText("Инициализация модели...");
                setIsProcessing(true);
            }
        } 
        else if (status === 'analyzing') {
            setLoadingText("Анализ изображения и описаний...");
            setProgress(100);
        }
        else if (status === 'complete') {
            setIsProcessing(false);
            setLoadingText("");
            // results - это массив { id, score } отсортированный
            const sortedIds = results.map((r: any) => r.id);
            
            // Фильтруем и сортируем mock данные согласно результатам ML
            const sortedProducts = sortedIds
                .map((id: number) => FURNITURE_MOCK.find(item => item.id === id))
                .filter((item: IFurniture | undefined) => item !== undefined) as IFurniture[];

            setDisplayedItems(sortedProducts);
            setSearchMode('image');
        }
        else if (status === 'error') {
            console.error(event.data.error);
            setIsProcessing(false);
            alert("Ошибка при обработке нейросетью. См. консоль.");
        }
    };

    return () => {
        workerRef.current?.terminate();
    };
  }, []);

  // Обычный текстовый поиск (фильтрация)
  useEffect(() => {
      if (searchMode === 'text') {
        const filtered = FURNITURE_MOCK.filter(item => 
            item.name.toLowerCase().includes(searchTerm.toLowerCase()) ||
            item.description.toLowerCase().includes(searchTerm.toLowerCase())
        );
        setDisplayedItems(filtered);
      }
  }, [searchTerm, searchMode]);

  // Сброс режима поиска при вводе текста
  const handleTextSearchChange = (val: string) => {
      setSearchMode('text');
      setSearchTerm(val);
  }

  // Обработчик загрузки картинки
  const handleImageSelect = (file: File) => {
      if (!workerRef.current) return;
      
      setSearchMode('image');
      setIsProcessing(true);
      setDisplayedItems([]); // Очищаем пока ищем
      setSearchTerm(`Поиск по фото: ${file.name}`);
      
      workerRef.current.postMessage({
        type: 'search',
        imageBlob: file,
        items: FURNITURE_MOCK
      });
  }

  return (
    <Container className="py-4">
      <BreadCrumbs crumbs={[{ label: "Каталог мебели" }]} />
      <h1 className="mb-4">Каталог</h1>
      
      <SearchBar 
        value={searchTerm} 
        setValue={handleTextSearchChange} 
        onImageSelect={handleImageSelect}
        isProcessing={isProcessing}
      />

      {isProcessing && (
        <div className="mb-3">
          <div className="mb-2">{loadingText}</div>
          <ProgressBar now={progress} label={`${Math.round(progress)}%`} animated />
        </div>
      )}

      <Row>
        {displayedItems.map((item) => (
          <Col key={item.id} md={4} lg={3} className="mb-4">
            <FurnitureCard
              {...item}
              onClick={(id) => navigate(`/catalog/${id}`)}
            />
          </Col>
        ))}
      </Row>
    </Container>
  );
};
```

### `src/workers/search.worker.ts`
Это основной файл логики поиска. Мы выносим тяжелые вычисления нейросети в Web Worker, чтобы не блокировать интерфейс. Разберем его по частям.

**1. Импорты и настройка**
Импортируем библиотеку `transformers` и настраиваем окружение. Мы используем модель SigLIP. Здесь также мы устанавливаем правило, что не ищем модели локальном сервере, а наоборот, скачиваем их с удаленного сервера. 
Также здесь мы указываем значение `TOP_K`, оно означает, сколько карточек будет выдаваться при поиске (по убыванию релевантности).

```typescript
import { 
    env, 
    AutoTokenizer, 
    AutoProcessor, 
    SiglipTextModel, 
    SiglipVisionModel,
    RawImage 
} from '@huggingface/transformers';
import { cosineSimilarity } from '../modules/math';

// НАСТРОЙКИ ОКРУЖЕНИЯ
env.allowLocalModels = false;
env.allowRemoteModels = true;

const MODEL_ID = 'Xenova/siglip-base-patch16-224'; 
const TOP_K = 3; 
```

**2. Сервис инициализации модели**
Создаем класс-синглтон для загрузки и хранения экземпляров модели, токенизатора и процессора. (Подробнее по ссылкам про <a href="https://huggingface.co/docs/transformers.js/api/models">модели</a>, <a href="https://huggingface.co/docs/transformers.js/api/tokenizers">токенизаторы</a>, <a href="https://huggingface.co/docs/transformers.js/api/processors">процессоры</a> в transformers.js).

В modelOptions мы задааем 2 параментра:
1) `device` - может быть как **wasm**, так и **webgpu**.
2) `dtype` - это квантизация модели. Значения могут быть: full-precision ("fp32"), half-precision ("fp16"), 8-bit ("q8", "int8", "uint8"), и 4-bit ("q4", "bnb4", "q4f16"). Подробно о том, что такое квантизация <a href="https://habr.com/ru/articles/887466/?ysclid=miw807a8w1320661335">тут</a>.


```typescript
class SiglipService {
    static tokenizer: any = null;
    static processor: any = null;
    static textModel: any = null;
    static visionModel: any = null;

    static async init(progress_callback?: (data: any) => void) {
        if (!this.tokenizer) {
            console.log(`Загрузка SigLIP (${MODEL_ID})...`);

            const modelOptions = {
                device: 'wasm',
                dtype: 'q8', 
            } as const;

            this.tokenizer = await AutoTokenizer.from_pretrained(MODEL_ID, { progress_callback });
            this.processor = await AutoProcessor.from_pretrained(MODEL_ID, { progress_callback });
            
            this.visionModel = await SiglipVisionModel.from_pretrained(MODEL_ID, {
                ...modelOptions,
                progress_callback
            });

            this.textModel = await SiglipTextModel.from_pretrained(MODEL_ID, {
                ...modelOptions,
                progress_callback
            });
        }
    }
}
```

**3. Обработка сообщений и изображений**
Воркер слушает сообщения. При получении картинки, он преобразует её в вектор (эмбеддинг) с помощью Vision Model.

```typescript
self.addEventListener('message', async (event) => {
    const { type, imageBlob, items } = event.data;

    if (type === 'search') {
        try {
            await SiglipService.init((data: any) => {
                self.postMessage({ status: 'progress', data });
            });

            self.postMessage({ status: 'analyzing' });

            // 1. ОБРАБОТКА КАРТИНКИ
            const imageUrl = URL.createObjectURL(imageBlob);
            const image = await RawImage.read(imageUrl);
            
            const image_inputs = await SiglipService.processor(image);
            const { pooler_output: imageOutput } = await SiglipService.visionModel(image_inputs);
            const imageVector = imageOutput.data; 
```

**4. Обработка текста и сравнение**
Далее мы берем описания товаров, превращаем их в векторы с помощью Text Model и сравниваем с вектором картинки через косинусное сходство.

```typescript
            // 2. ОБРАБОТКА ТЕКСТА
            const descriptions = items.map((item: any) => item.description);
            
            const text_inputs = await SiglipService.tokenizer(descriptions, { 
                padding: true, 
                truncation: true 
            });

            const { pooler_output: textOutput } = await SiglipService.textModel(text_inputs);
            
            // 3. СРАВНЕНИЕ
            const embeddingSize = 768; 
            const scores: { id: number, score: number, name: string }[] = [];

            for (let i = 0; i < items.length; i++) {
                const start = i * embeddingSize;
                const end = start + embeddingSize;
                const textVector = textOutput.data.slice(start, end);

                const sim = cosineSimilarity(Array.from(imageVector), Array.from(textVector));
                
                scores.push({ 
                    id: items[i].id, 
                    score: sim,
                    name: items[i].name 
                });
            }
```

**5. Завершение**
Сортируем результаты по убыванию сходства и отправляем обратно в основной поток.

```typescript
            // 4. Сортировка и Выдача
            scores.sort((a, b) => b.score - a.score);
            const results = scores.slice(0, TOP_K);

            self.postMessage({ status: 'complete', results });

            URL.revokeObjectURL(imageUrl);

        } catch (error) {
            console.error("Worker Error:", error);
            self.postMessage({ status: 'error', error: String(error) });
        }
    }
});
```

**Полный код `src/workers/search.worker.ts`:**

```typescript
import { 
    env, 
    AutoTokenizer, 
    AutoProcessor, 
    SiglipTextModel, 
    SiglipVisionModel,
    RawImage 
} from '@huggingface/transformers';
import { cosineSimilarity } from '../modules/math';

// НАСТРОЙКИ ОКРУЖЕНИЯ
env.allowLocalModels = false;
env.allowRemoteModels = true;

const MODEL_ID = 'Xenova/siglip-base-patch16-224'; 
const TOP_K = 3; 

class SiglipService {
    static tokenizer: any = null;
    static processor: any = null;
    static textModel: any = null;
    static visionModel: any = null;

    static async init(progress_callback?: (data: any) => void) {
        if (!this.tokenizer) {
            console.log(`Загрузка SigLIP (${MODEL_ID})...`);

            const modelOptions = {
                device: 'wasm',
                dtype: 'q8', 
            } as const;

            this.tokenizer = await AutoTokenizer.from_pretrained(MODEL_ID, { progress_callback });
            this.processor = await AutoProcessor.from_pretrained(MODEL_ID, { progress_callback });
            
            this.visionModel = await SiglipVisionModel.from_pretrained(MODEL_ID, {
                ...modelOptions,
                progress_callback
            });

            this.textModel = await SiglipTextModel.from_pretrained(MODEL_ID, {
                ...modelOptions,
                progress_callback
            });
        }
    }
}

self.addEventListener('message', async (event) => {
    const { type, imageBlob, items } = event.data;

    if (type === 'search') {
        try {
            await SiglipService.init((data: any) => {
                self.postMessage({ status: 'progress', data });
            });

            self.postMessage({ status: 'analyzing' });

            // 1. ОБРАБОТКА КАРТИНКИ
            const imageUrl = URL.createObjectURL(imageBlob);
            const image = await RawImage.read(imageUrl);
            
            const image_inputs = await SiglipService.processor(image);
            const { pooler_output: imageOutput } = await SiglipService.visionModel(image_inputs);
            const imageVector = imageOutput.data; 

            // 2. ОБРАБОТКА ТЕКСТА
            const descriptions = items.map((item: any) => item.description);
            
            const text_inputs = await SiglipService.tokenizer(descriptions, { 
                padding: true, 
                truncation: true 
            });

            const { pooler_output: textOutput } = await SiglipService.textModel(text_inputs);
            
            // 3. СРАВНЕНИЕ
            const embeddingSize = 768; 
            const scores: { id: number, score: number, name: string }[] = [];

            for (let i = 0; i < items.length; i++) {
                const start = i * embeddingSize;
                const end = start + embeddingSize;
                const textVector = textOutput.data.slice(start, end);

                const sim = cosineSimilarity(Array.from(imageVector), Array.from(textVector));
                
                scores.push({ 
                    id: items[i].id, 
                    score: sim,
                    name: items[i].name 
                });
            }

            console.log("Результаты сходства:", scores.sort((a, b) => b.score - a.score));

            // 4. Сортировка и Выдача
            scores.sort((a, b) => b.score - a.score);
            const results = scores.slice(0, TOP_K);

            self.postMessage({ status: 'complete', results });

            URL.revokeObjectURL(imageUrl);

        } catch (error) {
            console.error("Worker Error:", error);
            self.postMessage({ status: 'error', error: String(error) });
        }
    }
});
```
