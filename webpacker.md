Webpacker
=========

Это руководство покажет, как установить и использовать Webpacker для упаковки JavaScript, CSS и других ассетов для клиентов вашего приложения Rails, но учтите, что [от Webpacker отказались](https://github.com/rails/webpacker#webpacker-has-been-retired-)

После его прочтения вы узнаете:

* Что делает Webpacker, и почему он отличается от Sprockets.
* Как установить Webpacker и интегрировать его с выбранным фреймворком.
* Как использовать Webpacker для ассетов JavaScript.
* Как использовать Webpacker для ассетов CSS.
* Как использовать Webpacker для статичных ассетов.
* Как развернуть сайт, использующий Webpacker.
* Как использовать Webpacker в контекстах, альтернативных Rails, таких как engine или контейнеры Docker.

--------------------------------------------------------------

Что такое Webpacker?
--------------------

Webpacker это обертка Rails вокруг системы сборки [webpack](https://webpack.js.org), предоставляющий стандартную конфигурацию webpack и приемлемые значения по умолчанию.

### Что такое webpack?

Цель webpack, или любой другой системы сборки для фронтенд, в том, чтобы позволить писать код фронтенд удобным для разработчиков способом, а затем упаковывать этот код способом, удобным для браузеров. С помощью webpack можно управлять JavaScript, CSS и статичными ассетами, такими как изображения или шрифты. Webpack позволит писать свой код, ссылаться на другой код в вашем приложении, преобразовывать свой код и объединять свой код в легко скачиваемые пакеты.

Подробнее смотрите в [документации webpack](https://webpack.js.org).

### В чем отличие Webpacker от Sprockets?

Rails также поставляется со Sprockets, инструментом упаковки ассетов, чьи особенности пересекаются с Webpacker. Оба инструмента скомпилируют ваш JavaScript в удобные для браузера файлы, а также уменьшат размер и сделают их уникальными в production. В среде development Sprockets и Webpacker позволяют вам постепенно изменять файлы.

Sprockets, который был разработан для использования с Rails, немного проще в интеграции. В частности код можно добавить в Sprockets с помощью гема Ruby. Однако, webpack лучше для интеграции с большим количеством современных инструментов JavaScript и пакетов NPM, и позволяет более широкий диапазон интеграции. Новые приложения Rails настроены использовать webpack для JavaScript и Sprockets для CSS, хотя вы можете создавать CSS в webpack.

Следует выбрать Webpacker над Sprockets в новом проекте, если хотите использовать пакеты NPM, и/или хотите доступ к большинству современных особенностей и инструментов JavaScript. Следует выбрать Sprockets над Webpacker для старых приложений, где миграция может быть затратной, если хотите интегрировать с помощью гемов, или если у вас очень мало кода для упаковки.

Если вы знакомы со Sprockets, следующее руководство может подсказать вам как преобразовывать. Отметьте, что у каждого инструмента несколько разная структура, и концепции не всегда могут быть соотнесены напрямую.

|Задача                  | Sprockets            | Webpacker          |
|------------------------|----------------------|--------------------|
|Присоединить JavaScript |javascript_include_tag|javascript_pack_tag |
|Присоединить CSS        |stylesheet_link_tag   |stylesheet_pack_tag |
|Ссылка на изображение   |image_url             |image_pack_tag      |
|Ссылка на ассет         |asset_url             |asset_pack_tag      |
|Затребовать скрипт      |//= require           |import или require  |

Установка Webpacker
-------------------

Для использования Webpacker необходимо установить менеджер пакетов Yarn версии 1.x или выше, и необходим установленный Node.js версии 10.13.0 и выше.

NOTE: Webpacker зависит от NPM и Yarn. NPM, реестр менеджера пакетов Node, это основной репозиторий для публикации и скачивания проектов JavaScript с открытым кодом для запуска в Node.js или браузере. Это аналог rubygems.org для гемов Ruby. Yarn это утилита командной строки, позволяющая установку и управление зависимостей JavaScript, очень похожее на то, что Bundler делает для Ruby.

Чтобы включить Webpacker в новом проекте, добавьте `--webpack` в команду `rails new`. Чтобы добавить Webpacker в существующий проект, добавьте гем `webpacker` в `Gemfile` проекта, запустите `bundle install` а затем запустите `bin/rails webpacker:install`.

Установка Webpacker создает следующие локальные файлы:

|Файл                   |Размещение              |Пояснение                                                                                             |
|-----------------------|------------------------|------------------------------------------------------------------------------------------------------|
|Папка JavaScript       | `app/javascript`       |Место для исходников вашего фронтенд                                                                  |
|Конфигурация Webpacker | `config/webpacker.yml` |Настройки для гема Webpacker                                                                          |
|Конфигурация Babel     | `babel.config.js`      |Настройки для компилятора JavaScript [Babel](https://babeljs.io)                                      |
|Конфигурация PostCSS   | `postcss.config.js`    |Настройки для постпроцессора CSS [PostCSS](https://postcss.org)                                       |
|Browserlist            | `.browserslistrc`      |[Browserlist](https://github.com/browserslist/browserslist) управляет конфигурацией целевых браузеров |

Установка также вызывает менеджер пакетов `yarn`, создает файл `package.json` с базовым набором перечисленных пакетов и использует Yarn для установки этих зависимостей.

Использование
-------------

### Использование Webpacker для JavaScript

С установленным Webpacker, любой файл JavaScript в директории `app/javascripts/packs` по умолчанию будет скомпилирован в свой отдельный файл пакета.

Таким образом, если есть файл по имени `app/javascript/packs/application.js`, Webpacker создаст пакет, названный `application`, и его можно добавить в приложение Rails с помощью кода `<%= javascript_pack_tag "application" %>`. Для этого, в development Rails будет перекомпилировать файл `application.js` каждый раз, как он изменяется, и вы загружаете страницу, использующую этот пакет. Обычно файлом в фактической директории `packs` будет манифестом, в основном загружающим другие файлы, но он также может содержать произвольный код JavaScript.

Пакет по умолчанию, создаваемый Webpacker, будет ссылаться на JavaScript пакеты Rails по умолчанию, если они были включены в этом проекте:

```
import Rails from "@rails/ujs"
import Turbolinks from "turbolinks"
import * as ActiveStorage from "@rails/activestorage"
import "channels"

Rails.start()
Turbolinks.start()
ActiveStorage.start()
```

Вам будет нужно включить пакет, требующие эти пакеты, чтобы использовать их в вашем приложении Rails.

Важно отметить, что в директорию `app/javascript/packs` следует помещать только конечные точки доступа webpack; webpack создаст отдельный граф зависимостей для каждой конечной точки, поэтому большое количество пакетов увеличит нагрузку на компиляцию. Остальной исходный код ваших ассетов должен находиться вне этой директории, хотя Webpacker не накладывает каких-либо ограничений, и не делает каких-либо рекомендаций, как структурировать ваш исходный код. Вот пример:

```sh
app/javascript:
  ├── packs:
  │   # тут только файлы конечных точек доступа webpack
  │   └── application.js
  │   └── application.css
  └── src:
  │   └── my_component.js
  └── stylesheets:
  │   └── my_styles.css
  └── images:
      └── logo.svg
```

Обычно сам файл пакета это, в значительной степени, манифест, использующий `import` или `require` для загрузки необходимых файлов, а также может выполнять некоторую инициализацию.

Если хотите изменить эти директории, можно изменить `source_path` (по умолчанию `app/javascript`) и `source_entry_path` (по умолчанию `packs`) в файле `config/webpacker.yml`.

В файлах исходного кода, выражения `import` разрешаются относительно файла, выполняющего импорт, таким образом `import Bar from "./foo"` находит файл `foo.js` в той же директории, что и текущий файл, а `import Bar from "../src/foo"` находит файл в соседней директории с именем `src`.

### Использование Webpacker для CSS

Из коробки Webpacker поддерживает CSS и SCSS с помощью процессора PostCSS.

Чтобы включить код CSS в ваши пакеты, сначала включите ваши файлы CSS в файл пакета верхнего уровня, как будто это файл JavaScript. Таким образом, если ваш манифест CSS верхнего уровня в `app/javascript/styles/styles.scss`, можно импортировать его с помощью `import styles/styles`. Это сообщит webpack включить ваш файл CSS в скачивание. Чтобы фактически загрузить его на странице, включите `<%= stylesheet_pack_tag "application" %>` во вью, где `application` это то же самое имя пакета, что вы использовали.

Если вы используете фреймворк CSS, его можно добавить в Webpacker, следуя инструкциям загрузки фреймворка как модуля NPM с помощью `yarn`, обычно `yarn add <framework>`. У фреймворка должны быть инструкции по импортированию его в файлы CSS или SCSS.

### Использование Webpacker для статичных ассетов

По умолчанию [конфигурация Webpacker](https://github.com/rails/webpacker/blob/master/lib/install/config/webpacker.yml#L21) должна работать из коробки для статичных ассетов. Эта конфигурация включает несколько расширений изображений и шрифтов, позволяя webpack включать их в сгенерированный файл `manifest.json`.

С помощью webpack, статичные ассеты можно импортировать непосредственно в файлы JavaScript. Импортированное значение представляет URL ассета. Например:

```javascript
import myImageUrl from '../images/my-image.jpg'

// ...
let myImage = new Image();
myImage.src = myImageUrl;
myImage.alt = "I'm a Webpacker-bundled image";
document.body.appendChild(myImage);
```

Если нужно сослаться на статичные ассеты Webpacker из вью Rails, нужно явно затребовать эти ассеты из Webpacker-укомплектованных файлов JavaScript. В отличие от Sprockets, Webpacker не импортирует ваши статичные ассеты по умолчанию. Файл по умолчанию `app/javascript/packs/application.js` содержит шаблон для импортирования файлов из заданной директории, который можно откомментировать для каждой директории, в которой вы хотите хранить статичные файлы which you can uncomment for every directory you. Директории относительно `app/javascript`. Этот шаблон использует директорию `images`, но можно использовать что угодно в `app/javascript`:

```
const images = require.context("../images", true)
const imagePath = name => images(name, true)
```

Статичные ассеты будут выведены в директорию внутри `public/packs/media`. Например, изображение, расположенное и импортированное в `app/javascript/images/my-image.jpg`, будет выведено в `public/packs/media/images/my-image-abcd1234.jpg`. Чтобы отрендерить тег изображения для этого изображения во вью Rails, используйте `image_pack_tag 'media/images/my-image.jpg`.

Хелперы Webpacker ActionView для статичных файлов соответствуют хелперам файлопровода согласно следующей таблице:

|хелпер ActionView | хелпер Webpacker |
|------------------|------------------|
|favicon_link_tag  |favicon_pack_tag  |
|image_tag         |image_pack_tag    |

Также общий хелпер `asset_pack_path` принимает локальное расположение файла и возвращает его размещение webpacker для использования во вью Rails.

Также можно получить доступ к изображению, непосредственно сославшись на файл из файла CSS в `app/javascript`.

### Webpacker в Rails Engine

Начиная с 6 версии, Webpacker больше не осведомлен об engine, что означает, что у Webpacker нет схожих со Sprockets особенностей для использования в Rails engine.

Авторам гемов Rails engine, желающие поддержать потребителей, использующих Webpacker, рекомендуется распространять ассеты для фронтенда как пакет NPM в дополнение к самому гему и предоставлять инструкции (или установщик) для демонстрации как следует интегрировать в основное приложение. Хорошим примером этого подхода является [Alchemy CMS](https://github.com/AlchemyCMS/alchemy_cms).

### Замена модуля на лету (Hot Module Replacement, HMR)

Webpacker поддерживает HMR из коробки с помощью webpack-dev-server, и его можно переключать, установив опцию dev_server/hmr в webpacker.yml.

За подробностями обратитесь к [документации webpack на DevServer](https://webpack.js.org/configuration/dev-server/#devserver-hot).

Для поддержки HMR с React необходимо добавить react-hot-loader. Обратитесь к [руководству загрузки React на лету для начинающих](https://gaearon.github.io/react-hot-loader/getstarted/).

Не забудьте отключить HMR, если вы не запускаете webpack-dev-server, в противном случае вы получите "not found error" для таблиц стилей.

Webpacker в различных средах
----------------------------

В Webpacker по умолчанию есть три среды, `development`, `test` и `production`. Можно добавить дополнительные конфигурации сред в файле `webpacker.yml`, и установить разные значения по умолчанию для каждой среды, Webpacker также загрузит файл `config/webpack/<environment>.js` для дополнительной настройки среды.

## Запуск Webpacker в Development

Webpacker поставляется с двумя исполняемыми файлами для запуска в development: `./bin/webpack` и `./bin/webpack-dev-server`. Оба являются всего обертками вокруг стандартных исполняемых `webpack.js` и `webpack-dev-server.js`, и убеждаются, что загружаются правильные конфигурационные файлы и переменные среды, основываясь на вашей среде.

По умолчанию Webpacker компилирует автоматически по требованию в development, когда загружается страница Rails. Это означает, что вам не нужно запускать какой-то отдельный процесс, и что ошибки компиляции будут логированы в стандартный лог Rails. Это можно изменить, поменяв на `compile: false` в файле `config/webpacker.yml`. Запуск `bin/webpack` принудительно скомпилирует ваши пакеты.

Если хотите использовать перезагрузку кода в реальном времени, или если имеется столько JavaScript, что компиляция по запросу слишком медленная, необходимо запустить `./bin/webpack-dev-server` или `ruby ./bin/webpack-dev-server`. Этот процесс будет наблюдать за изменениями в файлах `app/javascript/packs/*.js` и автоматически перекомпилировать и перезагружать соответствующий браузер.

Пользователям Windows нужно запускать эти команды в терминале, отдельном от `bundle exec rails server`.

Как только вы запустили этот сервер разработки, Webpacker автоматически начнет проксирование всех запросов ассетов webpack к этому серверу. Когда вы остановите сервер, он вернет компиляцию по запросу.

[Документация Webpacker](https://github.com/rails/webpacker) предоставляет информацию по переменным среды, которые можно использовать для контроля над `webpack-dev-server`. Обратитесь к дополнительным заметкам в [документации rails/webpacker по использованию webpack-dev-server](https://github.com/rails/webpacker#development).

### Развертывание Webpacker

Webpacker добавляет задачу `webpacker:compile` в задачу rake `assets:precompile`, таким образом любой процесс развертывания, использующий `assets:precompile`, должен работать. Задача компиляции скомпилирует пакеты и разместит их в `public/packs`.

Дополнительная документация
---------------------------

Для подробностей по продвинутым темам, таким как использование Webpacker с популярными фреймворками, обратитесь к [документации Webpacker](https://github.com/rails/webpacker).
