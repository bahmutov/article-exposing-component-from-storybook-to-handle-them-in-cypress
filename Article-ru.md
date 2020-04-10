# Cypress + Storybook. Keeping test logic, data and component rendering in test. 
td; dr:
* Вы можете вынести ссылку на компонент из Storybook story чтобы протестировать его целиком силами Cypress (не разбивая логику теста на несколько частей).
* Cypress показался нашей команде настолько мощным инструментом, что мы оставили в прошлом React Testing Library, Enzyme и knobs.

Первые версии Cypress воспринимались как инструмент e2e-тестирования. Было любопытно наблюдать за ростом интереса front-end инженеров к теме, в которой всю жизнь правил Selenium. В то время типичное видео или статья, демонстрирующая возможности Cypress, ограничивались блужданием по случайно выбранному сайту и заслуженными лестными отзывами об API для ввода данных (прямо в JavaScript!).

Многие из нас догадались использовать Cypress для тестирования компонентов в изоляции предоставляемой такими средами как Storybook/Styleguidist/Docz. Хороший пример - статья Stefano Magni "Testing a Virtual List component with Cypress and Storybook". В ней предлагается создать Storybook Story, разместить в ней компонент и поместить в глобальную переменную данные, которые будут полезны для теста. Этот подход хорош, но в нём тест разрывается между Storybook и Cypress. Если у нас много компонентов, такие тесты будет сложно читать и поддерживать.

В этой статье я попытаюсь показать, как пойти чуть дальше и взять максимум от возможности писать на JavaScript в теле тестов Cypress. Для того чтобы увидеть как это работает, прошу загрузить исходный код по адресу https://github.com/daedalius/article-exposing-component-from-storybook-to-handle-them-in-cypress и выполнить команды **npm i** и **npn run test**.

Представим, что мы пишем адаптер для существующего компонента Datepicker и хотим покрыть его тестами. 

## Storybook
Со стороны Storybook всё, что нам нужно - пустая Story в которой в глобальной переменной сохраняется ссылка на тестируемый компонент. Чтобы не быть совсем бесполезной, эта Story нам отрисует один DOM-узел. Его роль - предоставить место под полигон, на котором Cypress будет тестировать целевой компонент.


```jsx
import React from 'react';
import Datepicker from './Datepicker.jsx';

export default {
  component: Datepicker,
  title: 'Datepicker',
};

export const emptyStory = () => {
    // Reference to retrieve it in Cypress during the test
    window.Datepicker = Datepicker;

    // Just a mount point
    return (
        <div id="component-test-mount-point"></div>
    )
};

```
Мы закончили со Storybook. Теперь переместим всё внимание на Cypress.

## Cypress
Я предпочитаю начинать работу над компонентом или покрытие его тестами с перечисления тест-кейсов. После того как мы определились с покрытием, получаем следующую заготовку под тест, который исполнит Cypress:

```jsx
/// <reference types="cypress" />

import React from 'react';
import ReactDOM from 'react-dom';

/**
 * <Datepicker />
 * * renders text field.
 * * renders desired placeholder text.
 * * renders chosen date.
 * * opens calendar after clicking on text field.
 */

context('<Datepicker />', () => {
    it('renders text field.', () => { });

    it('renders desired placeholder text.', () => { });

    it('renders chosen date.', () => { });

    it('opens calendar after clicking on text field.', () => { });
})
```

Для проведения теста нужна среда. Вспоминаем о только что развернутом Storybook. Перейдем напрямую к пустой Story, открыв её в новом окне кликнув по кнопке "Open canvas in new tab" на sidebar. Скопируем URL и нацелим туда Cypress:
```jsx
    const rootToMountSelector = '#component-test-mount-point';

    before(() => {
        cy.visit('http://localhost:12345/iframe.html?id=datepicker--empty-story');
        cy.get(rootToMountSelector);
    });
```

Как вы могли догадаться, мы будем рендерить интересующее нас состояние компонента в каждом тесте в одном и том же div. Чтобы тесты не влияли друг на друга, нужно размонтировать этот компонент после каждого теста. Добавим код очистки:
```jsx
    afterEach(() => {
        cy.document()
            .then((doc) => {
                ReactDOM.unmountComponentAtNode(doc.querySelector(rootToMountSelector));
            });
    });
```

Попробуем написать тест. Достанем ссылку на компонент и отрисуем его на странице:
```jsx
    const selectors = {
        innerInput: '.react-datepicker__input-container input',
    };

    it('renders text field.', () => {
        cy.window().then((win) => {
            ReactDOM.render(
                <win.Datepicker />,
                win.document.querySelector(rootToMountSelector)
            );
        });

        cy
            .get(selectors.innerInput)
            .should('be.visible');
    });
```

Вы чувствуете это? Ничто не останавливает нас передать в компонент любой props. Любое состояние. Любые данные. И всё теперь в одном месте - в Cypress!

## Тесты в несколько этапов, тестирование с обёрткой
Иногда компоненты содержат логику, которая исполняется при изменении props. Для примера возьмем \<Popup /\> c props по имени showed.
Когда showed=true, \<Popup /\> видим. При изменении showed c true на false, \<Popup /\> должен скрыться. Как это протестировать?

Такие задачи элементарно решаются императивно, однако в случае с декларативным React нам нужно что-то придумать. 
В нашей команде мы обычно создаём вспомогательный компонент со state. В данном случае state это boolean, отвечающий за showed props.
```jsx
let setPopupTestWrapperState = null;
const PopupTestWrapper = ({ showed, win }) => {
    const [isShown, setState] = React.useState(showed);
    setPopupTestWrapperState = setState;
    return <win.Popup showed={isShown} />
}
```
> Совет: Если hook у вам не завёлся (такое бывает) или вы против вызова setState извне компонента, перепишите на обычный class.

Применив написанную обёртку, завершим работу над тестом:
```jsx
it('becomes hidden after being shown when showed=false passed.', () => {
    // arrange
    cy.window().then((win) => {
        ReactDOM.render(
            <PopupTestWrapper
                showed={true}
                win={win}
            />,
            win.document.querySelector(rootToMountSelector)
        );
    });

    // act
    cy.then(() => { setPopupTestWrapperState(false); })

    // assert
    cy
        .get(selectors.popupWindow)
        .should('not.be.visible');
});
```

Вот и всё.

## Подытог: роли каждого из участников
Storybook:
* Поднимает stories содержащие собранные React компоненты для целей тестирования.
* Предоставляет реальную несинтетическую среду для исполнения тестов.
* Каждая story устанавливает глобальную ссылку на компонент в window (чтобы затем получить её в Cypress)
* Каждая story предоставляет точку монтирования, в которую затем будет рендерится компонент (при исполнении теста).
* Способен открыть каждый компонент в изоляции в чистой новой вкладке. Относитесь к этому как к испытательному полигону для тестовых Stories. А еще так удобно тестировать поведение на мобильных разрешениях.
> Совет: Используйте отдельный экземпляр Storybook для библиотеки компонентов. Не смешивайте тестовые stories с остальными.

Cypress:
* Содержит и запускает тесты
* Переходит к отдельным stories, получает ссылку на компонент.
* Отрисовывает компонент согласно логике теста с нужными данными и условиями (например, в мобильном разрешении).
* Взаимодействует с компонентом на странице.
* Предоставляет UI для визуализации процесса тестирования.

## Заключение
В этом разделе хотелось бы выразить личное мнение и позицию команды по некоторым вопросам, которые могли возникнуть у читателя. Написанное ниже не претендует на истину, может отличаться от реальности, а так же содержать арахис.

### На проекте я использую связку Jest с React Testing Library. В чем я себя ограничиваю?
* JSDOM это синтетическая среда, ограничивающая охват покрытия.  
* Не очень выходит работать с JSDOM так как это делал бы пользователь. Особенно когда речь заходит об имитации событий ввода.  
* Вы пишите юнит-тесты вслепую. Но зачем?  

### Стоит ли мне отказаться от связки Jest + React Testing Library?
Если вы воспринимаете тесты как среду для разработки - точно Да!  
Если вы воспринимаете тесты как показательную документацию - Да.  
Если вы пишете "низкоуровневые" юнит-тесты с покрытием деталей реализации и особенностей работы react-lifecycle - ... Не знаю. Я не пишу такой код. Вы уверены, что следуете первому принципу SOLID? Может быть, стоит что-то вынести из компонента и тестировать отдельно?  

### Почему бы просто не использовать cypress-react-unit-test? Зачем мне Storybook?
Вне сомнений - за этим подходом будущее.  
Но сейчас...  
Cypress имеет довольно хитрую архитектуру, запускающую собранный React и код в двух iframe. Иногда это ограничивает охват тестирования.  
Например: Представьте компонент который обращается к document.activeElement. При запуске в Cypress эта ссылка всегда будет указывать на document.body (из за особенностей iframe или нюансов реализации).
И это далеко не единственная проблема. Надеюсь, Gleb Bahmutov и команда Cypress все же справятся с этими трудностями 🤞
