Основною ціллю цього маніфесту є покращення якості проектів, які розробляються за допомогою фреймворку Meteor.
Нижче будуть наведені поширені помилки при роботі з фреймворком, поради та варіанти вирішення певних нетривіальних поблем.

#Безпека
####Приховання секретних ключів
Bad:
```js
S3.config = {
  key: '12345678901234567890',
  secret: '098765432109876543210987654321',
  bucket: 'top-secret-bucket'
};
```
Good:
```js
S3.config = {
  key: Meteor.settings.aws.key,
  secret: Meteor.settings.aws.secret,
  bucket: Meteor.settings.aws.bucket
};
```
config/production/settings.json
```json
{
  "public": {
    "aws": {
      "bucketUrl": "https://my-meteor-example.s3.amazonaws.com",
    }
  },
  "aws": {
    "accessKey": "12345678901234567890",
    "secret": "098765432109876543210987654321",
    "bucket": "top-secret-bucket"
  }   
}
```

####Приховання серверного коду
Bad:
both/accounts.js
```js
if (Meteor.isClient) {
  // do some stuff
}

if (Meteor.isServer) {
  ServiceConfiguration.configurations.insert({
    service: "instagram",
    clientId: Meteor.settings.instagram.clientId,
    secret: Meteor.settings.instagram.secret,
    scope: 'basic+likes+comments'
  });
}
```
Good:
client/accounts.js
```js
// do some stuff
```
server/accounts.js
```js
ServiceConfiguration.configurations.insert({
  service: "instagram",
  clientId: Meteor.settings.instagram.clientId,
  secret: Meteor.settings.instagram.secret,
  scope: 'basic+likes+comments'
});
```
##Обмеження доступу до БД
```
$ meteor remove autopublish & meteor remove insecure
```
####Приховання полів при публікації
Bad:
```js
Meteor.publish('users', function(limit) {
  return Meteor.users.find({}, {limit: limit}); 
);
```
Good:
```js
Meteor.publish('users', function(limit) {
  var projection = { 
    'username': 1,
    'createdAt': 1,
    'avatar': 1
  };
  return Meteor.users.find({}, {fields: projection, limit: limit}); 
);
```
####(?) Розширення документа поточного користувача
```js
Meteor.publish(null, function(limit) {
  if (this.userId) {
    var projection = { 
      'settings': 1,
      'roles': 1
    };
    return Meteor.users.find({}, {fields: projection, limit: limit}); 
  } else {
    this.ready();
  }
});
```

Публікація, ім'я якої встановлене як null, автоматично відправляє дані до всіх під'єднаних клієнтів.
```js
Meteor.publish(null, function() {
  // some stuff
});
```

####Валідація даних
Bad:
```js
// server
Meteor.methods({
  'messages/new': function (message) {
    message.createdAt = new Date;
    return Messages.insert(message);
  }
});
```
```js
// client
Template.Messages.events({
  'submit form': function (e) {
    var message;
    // ... формуєм повідомлення
    Meteor.call('messages/new', message);
  }
});
```
Bad:
```js
// server
Meteor.methods({
  'messages/new': function (message) {
    message.createdAt = new Date;
    return Messages.insert(message);
  }
});
```
```js
// client
Template.Messages.events({
  'submit form': function (e) {
    var message;
    // ... формуєм повідомлення
    check(message, { 
      userId: this.userId,
      conversationId: String,
      addresseeId : Match.Optional(String),
      text: Match.Optional(String),
      attachments: Match.Optional({
        images: Match.Optional([String]),
        docs: Match.Optional([String])
      }),
    });
    Meteor.call('messages/new', message);
  }
});
```
Good:
```js
//server
Meteor.methods({
  'messages/new': function (message) {
    check(message, { 
      userId: this.userId,
      conversationId: String,
      addresseeId : Match.Optional(String),
      text: Match.Optional(String),
      attachments: Match.Optional({
        images: Match.Optional([String]),
        docs: Match.Optional([String])
      }),
    });
    message.createdAt = new Date;
    return Messages.insert(message);
  }
});
```
```js
// client
Template.Messages.events({
  'submit form': function (e) {
    var message;
    // ... формуєм повідомлення
    Meteor.call('messages/new', message);
  }
});
```

#Рефакторинг
####Завжди при можливості поміщайте реактивні змінні в область видимості шаблона.
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
