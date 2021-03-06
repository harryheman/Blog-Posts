# `JavaScript` Audio Recording App :metal:

Let's develop an audio recording app.

Функционал приложения будет следующим:

- запись аудио;
- отображение записи с возможностью ее предварительного прослушивания и последующего сохранения или удаления;
- хранение аудио-файлов на сервере;
- извлечение аудио-файлов, хранящихся на сервере, и их отображение в браузере.

Основная технология, которую мы будем использовать, это [`MediaDevices`](https://developer.mozilla.org/ru/docs/Web/API/MediaDevices). Данная технология входит в состав глобального объекта [`Navigator`](https://developer.mozilla.org/ru/docs/Web/API/Navigator). Основным методом, предоставляемым указанным интерфейсом является [`getUserMedia()`](https://developer.mozilla.org/ru/docs/Web/API/MediaDevices/getUserMedia). Запись данных (в простых случаях) выполняется с помощью интерфейса [`MediaRecorder`](https://developer.mozilla.org/ru/docs/Web/API/MediaRecorder).

Интерфейс `MediaDevices` на сегодняшний день [поддерживается всеми современными браузерами](https://caniuse.com/mdn-api_mediadevices).

Для небольшой стилизации приложения будет использоваться [Sass](https://sass-lang.com/).

Выглядеть приложение будет примерно так:

<img src="https://habrastorage.org/webt/ir/jf/wt/irjfwtcq6wdrpmsmqdakiy4riig.png" />
<br />

Основным источником вдохновения при подготовке туториала для меня послужила [эта замечательная статья](https://www.sitepoint.com/mediastream-api-record-audio/).

Вы готовы? Тогда вперед!
<cut />
Начнем с разработки сервера.

## Сервер

Создаем директорию для проекта, переходим в нее, инициализируем проект и устанавливаем необходимые зависимости:

```bash
# создаем директорию
mkdir record-app
# переходим в нее
cd !$

# инициализируем проект
# `-y` означает выбор значений по умолчанию
yarn init -y
# или
npm init -y

# устанавливаем основные зависимости
yarn add express multer
# или
npm i ...

# устанавливаем зависимости для разработки
yarn add concurrently nodemon open-cli sass
```

Зависимости:

- [`express`](https://expressjs.com/ru/) - фреймворк для `Node.js`, облегчающий разработку `REST API`
- [`multer`](https://github.com/expressjs/multer/blob/master/doc/README-ru.md) - обертка над [`busboy`](https://www.npmjs.com/package/busboy) для разбора данных в формате `multipart/form-data`, часто используемая для сохранения файлов
- [`concurrently`](https://www.npmjs.com/package/concurrently) - утилита для одновременного выполнения команд, определенных в `package.json`
- [`nodemon`](https://www.npmjs.com/package/nodemon) - утилита для запуска сервера для разработки (сервера, который автоматически перезапускается при изменении наблюдаемых файлов)
- [`open-cli`](https://www.npmjs.com/package/open-cli) - утилита для автоматического открытия вкладки браузера по указанному адресу
- [`sass`](https://www.npmjs.com/package/sass) - утилита для преобразования `SASS` в `CSS`

Определяем команды для запуска сервера для разработки в `package.json`:

```json
"scripts": {
 "sass": "sass --no-source-map --watch public/style.scss:public/style.css",
 "server": "open-cli http://localhost:3000 && nodemon index.js",
 "start": "concurrently \"yarn sass\" \"yarn server\""
}
```

Команды:

- команда `sass` запускает утилиту для преобразования файла `public/style.scss` в файл `public/style.css` в режиме реального времени (`--watch`) и без создания карты источников (`--no-source-map`)
- команда `server` запускает сервер для разработки и открывает вкладку браузера по адресу `http://localhost:3000`
- команда `start` выполняет команды `sass` и `server`

В коде сервера мы будем использовать ES6-модули (`import/export`), поэтому определим в `package.json` тип основного файла сервера как модуль:

```json
"type": "module"
```

В качестве альтернативы для кода сервера можно использовать файл с расширением `.mjs`.

Структура проекта:

```
- record-app
 - node_modules
 - public - директория со статическими файлами (клиент)
   - img
     - microphone.png
     - pause.png
     - play.png
     - stop.png
   - index.html
   - script.js
   - style.scss
 - uploads - директория для аудио-файлов
 - index.js - сервер
 - package.json
```

Приступаем к реализации.

Что должен делать наш сервер? Многого от него не требуется: все, что он должен уметь - это обслуживать статические файлы из директорий `public` и `uploads`, а также сохранять и извлекать аудио-файлы по запросу клиента.

Импортируем необходимые модули:

```javascript
import express from 'express'
import { dirname } from 'path'
import { fileURLToPath } from 'url'
import { promises as fs } from 'fs'
import multer from 'multer'
```

Определяем абсолютный путь к текущей (рабочей) директории:

```javascript
const __dirname = dirname(fileURLToPath(import.meta.url))
```

Настраиваем `multer`:

```javascript
const upload = multer({
 storage: multer.diskStorage({
   // директория для файлов
   destination(req, file, cb) {
     cb(null, 'uploads/')
   },
   // названия файлов
   filename(req, file, cb) {
     cb(null, `${file.originalname}.mp3`)
   }
 })
})
```

Создаем приложение `express` и определяем директории со статическими файлами:

```javascript
const app = express()
app.use(express.static('public'))
app.use(express.static('uploads'))
```

Определяем маршрут для сохранения аудио с помощью `multer`, используемого в качестве посредника (middleware). Аудио будет отправляться методом `POST` в формате `multipart/form-data` по адресу `/save`:

```javascript
// поле, содержащее файл, должно называться `audio`
app.post('/save', upload.single('audio'), (req, res) => {
 // в ответ мы возвращаем статус `201`,
 // свидетельствующий об успешном сохранении файла на сервере
 res.sendStatus(201)
})
```

Определяем "роут" для извлечения сохраненных аудио и их отправки клиенту (метод - `GET`, адрес - `/records`):

```javascript
app.get('/records', async (req, res) => {
 try {
   // читаем содержимое директории `uploads`
   let files = await fs.readdir(`${__dirname}/uploads`)
   // нас интересуют только файлы с расширением `.mp3`
   files = files.filter((fileName) => fileName.split('.')[1] === 'mp3')
   // отправляем файлы клиенту
   res.status(200).json(files)
 } catch (e) {
   console.log(e)
 }
})
```

Наконец, определяем порт (по умолчанию `3000`) и запускаем сервер:

```javascript
const port = process.env.PORT || 3000
app.listen(port, () => {
 console.log('🚀')
})
```

На этом с сервером мы закончили. Переходим к клиенту.

## Клиент

Весь код клиента будет находиться в директории `public`.

Начнем с разметки (`index.html`).

```html
<head>
 <title>Record App</title>
 <!-- обратите внимание, что мы подключаем файл с расширением `.css` -->
 <!-- несмотря на то, что стили будем писать в файле с расширением `.scss` -->
 <link href="style.css" rel="stylesheet" />
</head>
<body>
 <header>
   <h1>Record App</h1>
 </header>

 <main>
   <!-- кнопка для начала записи -->
   <button class="btn" id="record_btn">
     <img src="img/microphone.png" alt="record" id="record_img" />
   </button>

   <!-- раздел для записей, полученных от сервера -->
   <section>
     <h2>My Records</h2>
     <div id="records_box"></div>
   </section>

   <!-- раздел для новой записи -->
   <section id="record_box">
     <h2>My Record</h2>
     <!-- контейнер для новой записи -->
     <div id="audio_box"></div>
     <!-- кнопки для сохранения и удаления новой записи -->
     <div class="action_box">
       <button class="btn btn_success" id="save_btn">Save</button>
       <button class="btn btn_danger" id="remove_btn">Remove</button>
     </div>
   </section>
 </main>

 <footer>&copy;&nbsp;</footer>

 <script src="script.js"></script>
</body>
```

_Обратите внимание_ на идентификаторы. Поскольку DOM-элементы с идентификаторами становятся свойствами глобального объекта `window`, мы сможем получать к ним доступ напрямую, т.е. без предварительного получения ссылки на элемент путем вызова таких методов как `querySelector()`.

Сделаем приложение красивым (`style.scss`):

```scss
// импортируем гугл-шрифт
@import url('https://fonts.googleapis.com/css2?family=Montserrat&display=swap');

// определяем переменные
// палитру я позаимствовал у `Bootstrap`
$primary: #0275d8;
$success: #5cb85c;
$info: #5bc0de;
$danger: #d9534f;
$dark: #292b2c;
$light: #f7f7f7;

// определяем миксин - "переиспользуемый" блок кода
@mixin flex-center {
 display: flex;
 justify-content: center;
 align-items: center;
}

// сброс стилей
* {
 margin: 0;
 padding: 0;
 box-sizing: border-box;
 font-family: 'Montserrat', sans-serif;
 font-size: 1rem;
 // применяем переменную
 color: $light;
}

// общие стили
body {
 // применяем миксин
 @include flex-center;
 flex-direction: column;
 min-height: 100vh;
 background-color: $dark;
 text-align: center;
}

// это позволяет прижать футер к нижней части области просмотра
main {
 flex: 1;
}

h1 {
 margin: 1rem 0;
 font-size: 1.8rem;
}

h2 {
 margin: 0.75rem 0;
 font-size: 1.4rem;
}

// стили для кнопок
.btn {
 padding: 0.5rem 1rem;
 border: none;
 outline: none;
 border-radius: 4px;
 font-weight: bold;
 letter-spacing: 1px;
 cursor: pointer;
 user-select: none;
 transition: 0.2s ease-in-out;

 // `&` - родительский селектор
 &_success {
   background-color: $success;
 }
 &_danger {
   background-color: $danger;
 }
 &:hover {
   background-color: $info;
   color: $dark;
 }
}

#record_btn {
 width: 100px;
 border-radius: 15%;
}

img {
 width: 100%;
}

// простая анимация
@keyframes show {
 from {
   opacity: 0;
   visibility: hidden;
   z-index: -1;
 }
 to {
   opacity: 1;
   visibility: visible;
   z-index: 1;
 }
}
@keyframes hide {
 from {
   opacity: 1;
   visibility: visible;
   z-index: 1;
 }
 to {
   opacity: 0;
   visibility: hidden;
   z-index: -1;
 }
}

// начальное состояние
#record_box {
 opacity: 0;
 visibility: hidden;
 z-index: -1;
}

// классы для анимации
.show {
 animation: show 0.4s linear forwards;
}

.hide {
 animation: hide 0.4s linear forwards;
}

.action_box {
 margin: 1rem 0;
}

#records_box {
 // снова применяем миксин - "переиспользуемость" в действии
 @include flex-center;
}

.audio_item {
 margin: 0.5rem;
 max-width: 320px;
 @include flex-center;

 // расширение общих стилей для кнопок за счет вложенности
 .btn {
   @include flex-center;
   width: 50px;
   background-color: $primary;
   margin-right: 0.5rem;
 }
}

footer {
 margin: 0.5rem 0;
}
```

Наконец, переходим к тому, ради чего мы, собственно, здесь собрались. Я, конечно же, имею ввиду клиентский скрипт. Но сначала кратко поговорим об интерфейсе `MediaDevices` и его методе `getUserMedia()`.

Интерфейс `MediaDevices` предоставляет доступ к медиа-устройствам пользователя, таким как камера, микрофон, а также к совместному использованию экрана. Разумеется, пользователь должен явно предоставить разрешение на такой доступ. Спецификация [`Media Capture and Streams`](https://w3c.github.io/mediacapture-main/) определяет довольно жесткие правила на этот счет.

Метод `getUserMedia()` запрашивает разрешение пользователя на использование медиа устройства (камера, микрофон). Результат возвращает промис, содержащий поток (объект `MediaStream`), который  состоит из треков (дорожек), содержащих требуемые данные. Данный метод принимает ограничения (объект `MediaStreamConstraints`), определяющий, какие типы медиа запрашиваются, а также требования для каждого типа.

В общем виде это выглядит так:

```javascript
// мы будем запрашивать разрешение на доступ только к аудио
const constraints = { audio: true, video: true }
let stream = null
navigator.mediaDevices.getUserMedia(constraints)
 .then((_stream) => { stream = _stream })
 // если возникла ошибка, значит, либо пользователь отказал в доступе,
 // либо запрашиваемое медиа-устройство не обнаружено
 .catch((err) => { console.error(`Not allowed or not found: ${err}`) })
```

`MediaRecorder` - это интерфейс `MediaStream Recording API` представляющий функциональность для простой записи медиа. Нас интересуют такие методы указанного интерфейса, как `start()` и `stop()`, свойство `state`, определяющее состояние записи, а также событие `dataavailable`, возникающее по окончанию записи.

Определяем глобальные переменные и забавы ради добавляем в подвал текущий год:

```javascript
let chunks = []
let mediaRecorder = null
let audioBlob = null

document.querySelector('footer').textContent += new Date().getFullYear()
```

Определяем утилиты для создания DOM-элементов и смены CSS-классов:

```javascript
// функция принимает объект с тегом создаваемого элемента,
// дочерними элементами в виде массива и атрибутами
const createEl = ({ tag = 'div', children, ...attrs }) => {
 // создаем элемент
 const el = document.createElement(tag)

 // если имеются атрибуты
 if (Object.keys(attrs).length > 0) {
   // добавляем их к элементу
   Object.entries(attrs).forEach(([attr, val]) => {
     el[attr] = val
   })
 }

 // если имеются дочерние элементы
 if (children) {
   // прибегаем к рекурсии
   children.forEach((_el) => {
     el.append(createEl(_el))
   })
 }

 // возвращаем элемент
 return el
}

// функция принимает элемент, новый и старый CSS-классы
const toggleClass = (el, oldC, newC) => {
 el.classList.remove(oldC)
 el.classList.add(newC)
}
```

Определяем функцию для начала записи:

```javascript
async function startRecord() {
 // проверяем поддержку
 if (!navigator.mediaDevices && !navigator.mediaDevices.getUserMedia) {
   return console.warn('Not supported')
 }

 // меняем изображение на основе состояния `mediaRecorder`
 record_img.src = `img/${
   mediaRecorder && mediaRecorder.state === 'recording' ? 'microphone' : 'stop'
 }.png`

 // если запись не запущена
 if (!mediaRecorder) {
   try {
     // получаем поток аудио-данных
     const stream = await navigator.mediaDevices.getUserMedia({
       audio: true
     })
     // создаем экземпляр `MediaRecorder`, передавая ему поток в качестве аргумента
     mediaRecorder = new MediaRecorder(stream)
     // запускаем запись
     mediaRecorder.start()
     // по окончанию записи и наличии данных добавляем эти данные в соответствующий массив
     mediaRecorder.ondataavailable = (e) => {
       chunks.push(e.data)
     }
     // обработчик окончания записи (см. ниже)
     mediaRecorder.onstop = mediaRecorderStop
   } catch (e) {
     console.error(e)
     record_img.src = ' img/microphone.png'
   }
 } else {
   // если запись запущена, останавливаем ее
   mediaRecorder.stop()
 }
}
```

Определяем функцию для обработки окончания записи:

```javascript
function mediaRecorderStop() {
 // если имеется предыдущая (новая) запись
 if (audio_box.children[0]?.localName === 'audio') {
   // удаляем ее
   audio_box.children[0].remove()
 }

 // создаем объект `Blob` с помощью соответствующего конструктора,
 // передавая ему `blobParts` в виде массива и настройки с типом создаваемого объекта
 // о том, что такое `Blob` и для чего он может использоваться
 // очень хорошо написано здесь: https://learn.javascript.ru/blob
 audioBlob = new Blob(chunks, { type: 'audio/mp3' })
 // метод `createObjectURL()` может использоваться для создания временных ссылок на файлы
 // данный метод "берет" `Blob` и создает уникальный `URL` для него в формате `blob:<origin>/<uuid>`
 const src = URL.createObjectURL(audioBlob)

 // создаем элемент `audio`
 const audioEl = createEl({ tag: 'audio', src, controls: true })

 audio_box.append(audioEl)
 // переключаем классы
 toggleClass(record_box, 'hide', 'show')

 // выполняем очистку
 mediaRecorder = null
 chunks = []
}
```

Определяем функцию для отправки новой записи на сервер для сохранения:

```javascript
async function saveRecord() {
 // данные должны иметь формат `multipart/form-data`
 const formData = new FormData()
 // запрашиваем у пользователя название для записи
 let audioName = prompt('Name?')
 // формируем название записи
 audioName = audioName ? Date.now() + '-' + audioName : Date.now()
 // первый аргумент - это название поля, которое должно совпадать
 // с названием поля в посреднике `upload.single()`
 formData.append('audio', audioBlob, audioName)

 try {
   await fetch('/save', {
     method: 'POST',
     body: formData
   })
   console.log('Saved')
   // сброс
   resetRecord()
   // получение записей от сервера
   fetchRecords()
 } catch (e) {
   console.error(e)
 }
}
```

Определяем функции сброса и удаления новой записи:

```javascript
function resetRecord() {
 toggleClass(record_box, 'show', 'hide')
 audioBlob = null
}

function removeRecord() {
 if (confirm('Sure?')) {
   resetRecord()
 }
}
```

Для формирования списка записей, полученных от сервера, нам потребуется функция для создания относительно сложного DOM-элемента. Для этого мы реализуем функцию высшего порядка на основе функции `createEl()`:

```javascript
const createRecordEl = (src) => {
 // получаем дату создания и название файла
 const [date, audioName] = src.replace('.mp3', '').split('-')
 // форматируем дату
 const audioDate = new Date(+date).toLocaleString()

 // создаем элемент
 return createEl({
   className: 'audio_item',
   children: [
     {
       tag: 'audio',
       src,
       // обработчик окончания воспроизведения записи
       onended: ({ currentTarget }) => {
         // меняем изображение
         currentTarget.parentElement.querySelector('img').src = 'img/play.png'
       }
     },
     {
       tag: 'button',
       className: 'btn',
       // обработчик нажатия кнопки для запуска воспроизведения (см. ниже)
       onclick: playRecord,
       children: [
         {
           tag: 'img',
           src: 'img/play.png'
         }
       ]
     },
     {
       tag: 'p',
       textContent: `${audioDate}${audioName ? ` - ${audioName}` : ''}`
     }
   ]
 })
}
```

Готовый элемент будет выглядеть примерно так:

```html
<div class="audio_item">
 <audio src="1632639698031-test1.mp3"></audio>
 <button class="btn">
   <img src="img/play.png">
 </button>
 <p>26.09.2021, 12:01:38 - test1</p>
</div>
```

Определяем функцию для получения записей от сервера и формирования соответствующего списка:

```javascript
async function fetchRecords() {
 try {
   // получаем файлы
   const files = await (await fetch('/records')).json()

   // очищаем контейнер
   records_box.innerHTML = ''

   // если имеется хотя бы один файл
   if (files.length > 0) {
     // формируем список
     files.forEach((file) => {
       records_box.append(createRecordEl(file))
     })
   // если файлов нет
   } else {
     // предлагаем пользователю что-нибудь записать
     records_box.append(
       createEl({
         tag: 'p',
         textContent: 'No records. Create one'
       })
     )
   }
 } catch (e) {
   console.error(e)
 }
}
```

Определяем функцию для запуска воспроизведения:

```javascript
function playRecord({ currentTarget: playBtn }) {
 // находим соответствующий элемент `audio`
 const audioEl = playBtn.previousElementSibling

 // если воспроизведение аудио еще не запускалось или было приостановлено
 if (audioEl.paused) {
   // запускаем воспроизведение
   audioEl.play()
   playBtn.firstElementChild.src = 'img/pause.png'
 } else {
   // останавливаем воспроизведение
   audioEl.pause()
   playBtn.firstElementChild.src = 'img/play.png'
 }
}
```

Наконец, добавляем кнопкам соответствующие обработчики и вызываем функцию для получения записей от сервера:

```javascript
record_btn.onclick = startRecord
save_btn.onclick = saveRecord
remove_btn.onclick = removeRecord

fetchRecords()
```

Для того чтобы убедиться в работоспособности приложения, выполняем команду `yarn start` или `npm start`. В директории `public` появляется файл `style.css`, сгенерированный на основе `style.scss`, запускается сервер для разработки (`🚀`) и в браузере открывается новая вкладка по адресу `http://localhost:3000`.

<img src="https://habrastorage.org/webt/h3/gi/85/h3gi85vrr98gttuhqzmny1up6w8.png" />
<br />

Нажатие кнопки приводит к запросу пользователя о выдаче разрешения на использование микрофона. Предоставляем программе такое разрешение. После этого начинается запись (об этом, в частности, свидетельствует красный индикатор рядом с кнопкой для закрытия вкладки и значок видеокамеры в адресной строке).

<img src="https://habrastorage.org/webt/xz/yf/an/xzyfanxm4os5stzxjqergzya1zi.png" />
<br />

Говорим что-нибудь в микрофон и нажимаем "Стоп". Появляется блок с новой записью и возможностью ее прослушивания, сохранения или удаления.

<img src="https://habrastorage.org/webt/8c/zx/sn/8czxsntplxvpy2o8-tla-qmoeyg.png" />
<br />

Нажимаем "Save" и вводим название записи. В директории `uploads` появляется новый файл в формате `.mp3`, в консоли появляется сообщение "Saved", список записей обновляется.

<img src="https://habrastorage.org/webt/vn/un/go/vnungosufttcneuduyhyqgjfc7y.png" />
<br />

The End.
