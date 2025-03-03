---
layout: post
title: Оптимизация загрузки и рендеринга веб-шрифтов
authors:
  - ilyagrigorik
date: 2019-08-16
updated: 2020-07-03
description: В этой публикации рассказывается, как загружать веб-шрифты, чтобы не возникали сдвиги макета и пустые страницы, если веб-шрифты недоступны во время загрузки страниц.
tags:
  - performance
  - fonts
feedback:
  - api
---

«Полный» веб-шрифт, включающий все стилистические варианты, которые могут вам не понадобиться, и все глифы, которые вы, возможно, не будете использовать, может иметь размер несколько мегабайтов. Прочитав эту публикацию, вы узнаете, как оптимизировать загрузку веб-шрифтов, чтобы для посетителей загружались только те данные, которые они будут использовать.

Чтобы решить проблему больших файлов, содержащих все варианты, специально разработано правило CSS `@font-face`, позволяющее разделить семейство шрифтов на коллекцию ресурсов. Например, на подмножества Unicode и на отдельные варианты стилей.

Учитывая такие объявления, браузер определяет перечень требуемых подмножеств и вариантов и загружает минимальный набор, необходимый для рендеринга текста, что очень удобно. Но если вы не будете аккуратны, могут возникнуть проблемы с производительностью на критически важном пути рендеринга и задержки при рендеринге текста.

### Поведение, используемое по умолчанию

Отложенная загрузка шрифтов имеет важное неявное последствие, которое может замедлять рендеринг текста: чтобы определить, какие ресурсы шрифта нужны для рендеринга текста, браузер должен [создать дерево рендеринга](https://developers.google.com/web/fundamentals/performance/critical-rendering-path/render-tree-construction), зависящее от деревьев DOM и CSSOM. В результате запросы шрифтов выполняются намного позже запросов других критических ресурсов, и браузер может заблокировать рендеринг текста до тех пор, пока не будет получен нужный ресурс.

{% Img src="image/admin/NgSTa9SirmikQAq1G5fN.png", alt="Критически важный путь рендеринга шрифта", width="800", height="303" %}

1. Браузер запрашивает HTML-документ.
2. Браузер начинает синтаксический анализ ответа HTML и создание модели DOM.
3. Браузер обнаруживает CSS, JS и другие ресурсы и отправляет запросы.
4. После получения всего содержимого CSS браузер создает CSSOM и объединяет его с деревом DOM, чтобы создать дерево рендеринга.
    - Запросы шрифтов отправляются после того, как по дереву рендеринга можно будет определить, какие варианты шрифтов необходимы для рендеринга указанного текста на странице.
5. Браузер выполняет разметку и отображает контент на экране.
    - Если шрифт еще недоступен, браузер не может выполнить рендеринг ни одного пикселя текста.
    - После того, как шрифт станет доступен, браузер отображает пиксели текста.

«Гонка» между первым отображением содержимого страницы, которое можно выполнить вскоре после создания дерева рендеринга, и запросом ресурса шрифта приводит к «проблеме пустого текста», когда браузер может отобразить макет страницы, но опускает весь текст.

Предварительно загружая веб-шрифты и используя свойство `font-display` для управления действиями браузеров в отношении недоступных шрифтов, можно предотвратить появление пустых страниц и сдвиги макета из-за загрузки шрифтов.

### Предварительная загрузка ресурсов веб-шрифтов

Если высока вероятность того, что на странице потребуется определенный веб-шрифт, размещенный по заранее известному URL-адресу, можно [настроить приоритеты ресурсов](https://developers.google.com/web/fundamentals/performance/resource-prioritization). С помощью элемента `<link rel="preload">` можно заблаговременно создать запрос веб-шрифта на ранней стадии критического пути рендеринга, не дожидаясь создания CSSOM.

### Настройка задержки рендеринга текста

Предварительная загрузка повышает вероятность того, что веб-шрифт будет доступен при рендеринге содержимого страницы, но она не гарантирует это. Вам все равно необходимо учитывать, каким образом браузеры ведут себя при рендеринге текста, в котором используется `семейство шрифтов`, которое еще недоступно.

В публикации «[Предотвращение невидимого текста во время загрузки шрифтов](/avoid-invisible-text/)» рассказывается, что по умолчанию браузеры ведут себя не всегда одинаково. Тем не менее с помощью свойства [`font-display`](https://developer.mozilla.org/docs/Web/CSS/@font-face/font-display) можно указывать современным браузерам, как они должны действовать.

Как и действия, связанные со временем ожидания существующих шрифтов и реализованные в некоторых браузерах, свойство `font-display` разделяет срок жизни операции загрузки шрифта на три указанных ниже основных периода.

1. Первый период — **период блокировки шрифта**. Если в течение этого периода начертание шрифта еще не загружено, любой элемент, пытающийся использовать этот шрифт, должен выполнять рендеринг невидимого резервного начертания шрифта. Если начертание шрифта будет успешно загружено в течение периода блокировки, оно будет использоваться как обычно.
2. **Период замены шрифта** наступает сразу после периода блокировки шрифта. Если в течение этого периода начертание шрифта не загружено, любой элемент, пытающийся его использовать, должен выполнять рендеринг резервного начертания шрифта. Если начертание шрифта будет успешно загружено в течение периода замены, оно будет использоваться как обычно.
3. **Период ошибки шрифта** наступает сразу после периода замены шрифта. Если на момент начала этого периода начертание шрифта еще не загружено, загрузка будет помечена как неудачная, и это приведет к штатному переходу на резервный шрифт. В противном случае начертание шрифта будет использоваться как обычно.

Понимая эти периоды, с помощью свойства `font-display` можно указать, каким образом следует выполнять рендеринг шрифта в зависимости от того, был ли он загружен, или от того, когда он был загружен.

Чтобы можно было работать со свойством `font-display`, добавьте его в правила `@font-face`:

```css
@font-face {
  font-family: 'Awesome Font';
  font-style: normal;
  font-weight: 400;
  font-display: auto; /* или block, swap, fallback, optional */
  src: local('Awesome Font'),
       url('/fonts/awesome-l.woff2') format('woff2'), /* будет предварительно загружен */
       url('/fonts/awesome-l.woff') format('woff'),
       url('/fonts/awesome-l.ttf') format('truetype'),
       url('/fonts/awesome-l.eot') format('embedded-opentype');
  unicode-range: U+000-5FF; /* Латинские глифы */
}
```

В настоящее время для свойства `font-display` поддерживаются указанные ниже значения.

- `auto`
- `block`
- `swap`
- `fallback`
- `optional`

Дополнительные сведения о предварительной загрузке шрифтов и свойстве `font-display` см. в указанных ниже публикациях.

- [Предотвращение невидимого текста во время загрузки шрифтов](/avoid-invisible-text/)
- [Управление производительностью шрифта с помощью свойства font-display](https://developers.google.com/web/updates/2016/02/font-display)
- [Предотвращение смещения макета и мигания невидимого текста (FOIT) путем предварительной загрузки дополнительных шрифтов](/preload-optional-fonts/)

### API загрузки шрифтов

Используя элемент `<link rel="preload">` вместе со свойством `font-display` в CSS, можно управлять загрузкой и рендерингом шрифтов без значительных дополнительных накладных расходов. Но если вам нужно выполнить дополнительные настройки и вы готовы нести накладные расходы, связанные с работой JavaScript, есть еще один вариант.

[API загрузки шрифтов](https://www.w3.org/TR/css-font-loading/) предоставляет интерфейс с поддержкой скриптов для определения начертаний шрифтов CSS и управления ими, отслеживания хода их загрузки и переопределения их поведения, используемого по умолчанию, при отложенной загрузке. Например, если вы уверены, что потребуется тот или иной вариант шрифта, вы можете определить его и указать браузеру инициировать немедленное получение ресурса шрифта:

```javascript
var font = new FontFace("Awesome Font", "url(/fonts/awesome.woff2)", {
  style: 'normal', unicodeRange: 'U+000-5FF', weight: '400'
});

// не ждать дерева рендеринга, инициировать немедленное получение!
font.load().then(function() {
  // применить шрифт (что может привести к повторному рендерингу текста и изменению макета страницы)
  // после окончания загрузки шрифта
  document.fonts.add(font);
  document.body.style.fontFamily = "Awesome Font, serif";

  // ИЛИ... по умолчанию содержимое скрыто;
  // оно отобразится после того как шрифт станет доступным
  var content = document.getElementById("content");
  content.style.visibility = "visible";

  // ИЛИ... примените собственную стратегию рендеринга здесь...
});
```

Более того, поскольку можно проверять статус шрифта (с помощью метода [`check()`](https://www.w3.org/TR/css-font-loading/#font-face-set-check)) и отслеживать ход его загрузки, вы можете задать пользовательскую стратегию рендеринга текста на страницах.

- Можно приостановить рендеринг всего текста, пока шрифт не станет доступным.
- Можно настроить отдельное время ожидания для каждого шрифта.
- Можно использовать резервный шрифт, чтобы разблокировать функцию рендеринга и добавить новый стиль, в котором используется нужный шрифт, после того, как этот шрифт станет доступным.

Лучше всего то, что можно сочетать указанные выше стратегии для различного контента на странице. Например, можно отложить рендеринг текста в некоторых разделах до тех пор, пока нужный шрифт не станет доступным, использовать резервный шрифт, а затем повторно выполнить рендеринг после завершения загрузки шрифта.

{% Aside %} API загрузки шрифтов [недоступен в браузерах старых версий](http://caniuse.com/#feat=font-loading). Похожие функции имеются в [полифиле FontLoader](https://github.com/bramstein/fontloader) и в [библиотеке WebFontloader](https://github.com/typekit/webfontloader), хотя при их использовании возникают еще большие накладные расходы из-за дополнительной зависимости JavaScript. {% endAside %}

### Необходимость правильного кеширования

Шрифты, как правило, представляют собой статические ресурсы, которые обновляются нечасто. В результате для них идеально подходят длительные сроки действия (max-age). Обязательно указывайте для всех ресурсов шрифтов [условный заголовок ETag](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching#validating-cached-responses-with-etags) и [оптимальную политику Cache-Control](https://developers.google.com/web/fundamentals/performance/optimizing-content-efficiency/http-caching#cache-control).

Если в вашем веб-приложении используется [служебный сценарий](https://developer.chrome.com/docs/workbox/service-worker-overview/), получение ресурсов шрифтов с применением [стратегии «сначала кэш»](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/#cache-then-network) подходит для большинства сценариев использования.

Не следует хранить шрифты с использованием свойства [`localStorage`](https://developer.mozilla.org/docs/Web/API/Window/localStorage) или API [IndexedDB](https://developer.mozilla.org/docs/Web/API/IndexedDB_API); у каждого из этих способов есть свой набор проблем с производительностью. Кэш HTTP браузера — лучший и самый надежный механизм доставки ресурсов шрифтов в браузер.

## Контрольный список загрузки веб-шрифтов

- **Настройте загрузку и рендеринг шрифта с помощью элемента `<link rel="preload">`, свойства `font-display` или API загрузки шрифтов:** действия функции отложенной загрузки, выполняемые по умолчанию, могут привести к задержке при рендеринге текста. Эти функции веб-платформы позволяют переопределить такое поведение для определенных шрифтов и указать пользовательские стратегии рендеринга и времени ожидания для различного содержимого на странице.
- **Укажите политики повторной проверки и оптимального кэширования:** шрифты — это статические ресурсы, которые обновляются нечасто. Сделайте так, чтобы ваши серверы предоставляли долговременную временную метку времени max-age с максимальным возрастом и токен повторной проверки, чтобы можно было эффективно использовать шрифты на разных страницах. При использовании служебного сценария подходит стратегия «сначала кэш».

## Автоматическая проверка поведения при загрузке веб-шрифта  с помощью Lighthouse

С помощью [Lighthouse](https://developers.google.com/web/tools/lighthouse) можно автоматизировать процесс проверки соответствия рекомендациям по оптимизации веб-шрифтов.

С помощью указанных ниже аудитов можно периодически проверять, соответствуют ли ваши страницы рекомендациям по оптимизации веб-шрифтов.

- [Preload key requests](/uses-rel-preload/)
- [Uses inefficient cache policy on static assets](/uses-long-cache-ttl/)
- [All text remains visible during WebFont loads](/font-display/)
