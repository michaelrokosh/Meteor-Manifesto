Основною ціллю цього маніфесту є покращення якості проектів, які розробляються за допомогою фреймворку Meteor.
Нижче будуть наведені поширені помилки при роботі з фреймворком, поради та варіанти вирішення певних нетривіальних поблем.

##### Завжди при можливості поміщайте реактивні змінні в область видимості шаблона.
```js
// Bad
Template.ColorScheme.events({
  'input .color': function (e) {
    Session.set('currentColor', e.target.value)
  }
});

Template.ColorScheme.helpers({
  currentColor: Session.get('currentColor')
});
```
```js
// Good
Template.ColorScheme.events({
  'input .color': function (e) {
    Template.instance().currentColor.set(e.target.value)
  }
});

Template.ColorScheme.helpers({
  currentColor: Template.instance().currentColor.get()
});

Template.ColorScheme.onCreated(function () {
  this.currentColor = new ReactiveVar();
});
```
