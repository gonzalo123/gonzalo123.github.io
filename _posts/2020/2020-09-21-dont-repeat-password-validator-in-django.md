---
layout: post
permalink: /:year/:month/:day/:title/
title: "Don't repeat password validator in Django"
date: "2020-09-21"
categories: 
  - "technology"
tags: 
  - "django"
  - "python"
coverImage: "120138.png"
---

Django has a set of default password validators by default in django.contrib.auth.password_validator

```python
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
```

Normally those validators aren't enough (at least for me) but it's very easy to create customs validators. There're also several validators that we can use, for example those [ones](https://sixfeetup.com/blog/custom-password-validators-in-django).

I normally need to avoid users to repeat passwords (for example the last ten ones). To do that we need to create a custom validator. Whit this validator we also need to create a model to store the last passwords (the hash). The idea is to persits the hash of the password each time the user changes the password. As well as is always a good practice not to use the default User model, we're going to create a CustomUser model

```python
class CustomUser(AbstractUser):
    def __init__(self, *args, **kwargs):
        super(CustomUser, self).__init__(*args, **kwargs)
        self.original_password = self.password

    def save(self, *args, **kwargs):
        super(CustomUser, self).save(*args, **kwargs)
        if self._password_has_been_changed():
            CustomUserPasswordHistory.remember_password(self)

    def _password_has_been_changed(self):
        return self.original_password != self.password
```

And now we can create our CustomUserPasswordHistory implementing the remember_password method.

```python
class CustomUserPasswordHistory(models.Model):
    username = models.ForeignKey(CustomUser, on_delete=models.CASCADE)
    old_pass = models.CharField(max_length=128)
    pass_date = models.DateTimeField()

    @classmethod
    def remember_password(cls, user):
        cls(username=user, old_pass=user.password, pass_date=localtime()).save()
```

Now the validator:

```python
class DontRepeatValidator:
    def __init__(self, history=10):
        self.history = history

    def validate(self, password, user=None):
        for last_pass in self._get_last_passwords(user):
            if check_password(password=password, encoded=last_pass):
                self._raise_validation_error()

    def get_help_text(self):
        return _("You cannot repeat passwords")

    def _raise_validation_error(self):
        raise ValidationError(
            _("This password has been used before."),
            code='password_has_been_used',
            params={'history': self.history},
        )

    def _get_last_passwords(self, user):
        all_history_user_passwords = CustomUserPasswordHistory.objects.filter(username_id=user).order_by('id')

        to_index = all_history_user_passwords.count() - self.history
        to_index = to_index if to_index > 0 else None
        if to_index:
            [u.delete() for u in all_history_user_passwords[0:to_index]]

        return [p.old_pass for p in all_history_user_passwords[to_index:]]
```

We can see how it works with the unit tests:

```python
class UserCreationTestCase(TestCase):
    def setUp(self):
        self.user = User.objects.create(username='gonzalo')

    def test_persist_password_to_history(self):
        self.user.set_password('pass1')
        self.user.save()

        all_history_user_passwords = CustomUserPasswordHistory.objects.filter(username_id=self.user)
        self.assertEqual(1, all_history_user_passwords.count())


class DontRepeatValidatorTestCase(TestCase):
    def setUp(self):
        self.user = User.objects.create(username='gonzalo')
        self.validator = DontRepeatValidator()

    def test_validator_with_new_pass(self):
        self.validator.validate('pass33', self.user)
        self.assertTrue(True)

    def test_validator_with_repeated_pass(self):
        for i in range(0, 11):
            self.user.set_password(f'pass{i}')
            self.user.save()

        with self.assertRaises(ValidationError):
            self.validator.validate('pass3', self.user)

    def test_keep_only_10_passwords(self):
        for i in range(0, 11):

            self.user.set_password(f'pass{i}')
            self.user.save()

        self.validator.validate('xxxx', self.user)

        all_history_user_passwords = CustomUserPasswordHistory.objects.filter(username_id=self.user)
        self.assertEqual(10, all_history_user_passwords.count())
```

Full source code in my [github](https://github.com/gonzalo123/dont-repeat-password-validator-Django)
