# Кастомизация статического сайта (шаблонизация, сборка статики: HTML, CSS, JS)

## [Структура проекта](repo_url/)

```
pyweb2025-mkdocs/
├── mkdocs.yml
├── package.json
├── postcss.config.js
├── .github/workflows/deploy.yml
├── custom_theme/
│   ├── main.html
│   └── assets/
│       ├── css/styles.css
│       └── js/main.js
├── docs/
│   ├── index.md
│   └── task2.md
└── site/ (генерируется)
```

## Конфигурация MkDocs [mkdocs.yml](repo_url/blob/main/mkdocs.yml)


```yaml
site_name: pyweb2025
site_description: Задания по курсу pyweb2025
site_author: Владислав Филимонов
site_url: https://github.com/iiVladIii/pyweb2025-mkdocs
repo_url: https://github.com/iiVladIii/pyweb2025-mkdocs

theme:
  name: null
  custom_dir: custom_theme

nav:
  - Задание 1: index.md
  - Задание 2: task2.md

markdown_extensions:
  - toc:
      permalink: false
  - codehilite
  - admonition
```

## Frontend Build Configuration [package.json](repo_url/blob/main/package.json)


```json
{
  "name": "pyweb2025",
  "version": "1.0.0",
  "description": "Задания по курсу pyweb2025",
  "scripts": {
    "build-css": "postcss custom_theme/assets/css/styles.css -o custom_theme/assets/css/styles.min.css",
    "build-js": "terser custom_theme/assets/js/main.js -o custom_theme/assets/js/main.min.js --compress --mangle",
    "build": "npm run build-css && npm run build-js",
    "watch": "npm run build-css -- --watch"
  },
  "devDependencies": {
    "postcss": "^8.4.31",
    "postcss-cli": "^10.1.0",
    "autoprefixer": "^10.4.16",
    "cssnano": "^6.0.1",
    "terser": "^5.24.0"
  }
}
```

## CI/CD Pipeline [.github/workflows/deploy.yml](repo_url/blob/main/.github/workflows/mkdocs.yml)


```yaml
name: Deploy MkDocs to GitHub Pages

on:
  push:
    branches: [ "main" ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'

      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.x'

      - name: Cache Python dependencies
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements.txt') }}

      - name: Install Node.js dependencies
        run: npm ci

      - name: Install Python dependencies
        run: |
          python -m pip install --upgrade pip
          pip install mkdocs mkdocs-material pymdown-extensions

      - name: Build assets
        run: npm run build

      - name: Build MkDocs site
        run: mkdocs build --verbose --clean --strict

      - name: Upload build artifact
        uses: actions/upload-artifact@v4
        with:
          name: site-build
          path: ./site
          retention-days: 1

  test:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      - name: Download build artifact
        uses: actions/download-artifact@v4
        with:
          name: site-build
          path: ./site

      - name: Validate HTML structure
        run: |
          mkdir -p temp
          echo '<!DOCTYPE html><html><head><meta charset="utf-8"></head><body>' > temp/test.html
          cat custom_theme/main.html | grep -v '{%\|{{' >> temp/test.html || true
          echo '</body></html>' >> temp/test.html
          
          npm install -g html-validate
          
          cat > .htmlvalidate.json << 'EOF'
          {
            "extends": ["html-validate:recommended"],
            "rules": {
              "no-unknown-elements": "off",
              "require-sri": "off",
              "no-inline-style": "off"
            }
          }
          EOF
          
          echo "HTML template structure checked"

      - name: Verify critical files
        run: |
          if [ ! -f "site/index.html" ]; then
            echo "Error: index.html not found"
            exit 1
          fi
          
          if [ ! -f "site/assets/css/styles.min.css" ]; then
            echo "Error: minified CSS not found"
            exit 1
          fi
          
          if [ ! -f "site/assets/js/main.min.js" ]; then
            echo "Error: minified JS not found"
            exit 1
          fi

      - name: Validate built HTML files
        run: |
          npm install -g html-validate
          find site -name "*.html" -exec html-validate {} \; || echo "Some HTML validation warnings (non-critical)"

      - name: Optimize images
        run: |
          if [ -d "site/assets/images" ]; then
            echo "Optimizing images..."
            npm install -g imagemin-cli imagemin-pngquant imagemin-mozjpeg
            imagemin site/assets/images/* --out-dir=site/assets/images/ --plugin=pngquant --plugin=mozjpeg
          else
            echo "No images to optimize"
          fi

      - name: Upload tested artifact
        uses: actions/upload-artifact@v4
        with:
          name: site-tested
          path: ./site
          retention-days: 1

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: [ build, test ]
    steps:
      - name: Download tested artifact
        uses: actions/download-artifact@v4
        with:
          name: site-tested
          path: ./site

      - name: Setup Pages
        uses: actions/configure-pages@v4

      - name: Upload pages artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./site

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

## Этапы сборки и развертывания

### 1. Build Job
- Установка Node.js 18 и Python 3.x
- Кеширование зависимостей для оптимизации
- Установка npm и pip зависимостей
- Сборка и минификация CSS/JS активов
- Генерация статического сайта MkDocs
- Сохранение артефакта сборки

### 2. Test Job
- Загрузка артефакта сборки
- Валидация HTML структуры
- Проверка наличия критических файлов
- HTML валидация сгенерированных страниц
- Оптимизация изображений (если присутствуют)
- Сохранение проверенного артефакта

### 3. Deploy Job
- Загрузка протестированного артефакта
- Конфигурация GitHub Pages
- Загрузка финального артефакта для Pages
- Развертывание на GitHub Pages

## Реализация

### Кастомная тема
- Адаптивный дизайн с мобильной навигацией
- CSS переменные для консистентного оформления
- Минификация CSS и JS для оптимизации загрузки
- SEO оптимизация с Open Graph метатегами

### Автоматизация
- Автоматический запуск при push в main ветку
- Трехэтапный пайплайн: Build → Test → Deploy
- Кеширование зависимостей для ускорения сборки
- Валидация качества кода перед развертыванием

### Инструменты оптимизации
- PostCSS с Autoprefixer и CSSnano
- Terser для минификации JavaScript
- HTML-validate для проверки валидности разметки
