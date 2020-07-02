---
title: "Improving Django's PasswordResetConfirmView"
date: 2019-05-31T20:17:30-04:00
classes: wide
categories:
  - Django
tags:
  - Django
  - Python
  - Programming
  - IDOR
---

Django provides some fantastic helpers for handling user authentication, password resets and changes.

I've added password reset functionality to a few websites before and when adding the feature to a site running Django I found a small part of the built in PasswordResetConfirmView that I wasn't happy with. 

The default email that is sent to a user to reset their password contains a url that links to the password reset page. This url, whilst adequately secure, contains the base64 encoded id of the given user.

The problem with showing this id to the user is:

* It reveals to the user how many other users your site may have. (If their id is 350, they can guess you have at least 350 users)
* It provides to the user the primary key of their user account in the database which could aid in the exploitation of Insecure Direct Object Reference (IDOR).


## How to improve PasswordResetConfirmView

To improve the built in PasswordResetConfirmView we will need to extend the built in User model with a second Profile model.
There are other ways of extending the User model, but this way is simple, quick and fits most needs.
Most Django based websites handling user signups will probably need an extended User model anyway.

The code example (models.py) below sets up a minimal Profile model that is automatically created & updated with the built-in User model.
The Profile model also provides an account_id field which we will use to improve the built in password reset method.

#### models.py

```python
import uuid
from django.db import models
from django.db.models.signals import post_save
from django.contrib.auth.models import User
from django.dispatch import receiver


class Profile(models.Model):
  user = models.OneToOneField(User, on_delete=models.CASCADE)
  account_id = models.UUIDField(unique=True, default=uuid.uuid4)


@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)


@receiver(post_save, sender=User)
def save_user_profile(sender, instance, **kwargs):
    instance.profile.save()
```

Once we have the extended User model in place, we can use this to provide a more secure/appropriate ID for an individual user.

The account_id field on the Profile model is a uuid4 type which provides a high degree of randomness and doesn't reveal anything about how many users our site may have. We have set the field to be unique per user.

The heart of improving PasswordResetConfirmView is in providing our own that overrides the built in functionality.


In the file below (CustomPasswordResetConfirmView.py) I declare a view that overrides the built in view.
Normally PasswordResetConfirmView takes the base64 encoded user.pk passed to it in the url, decodes it and then looks the user up in the database.
With our custom view we are taking our account_id passed in the url, looking up the profile it belongs to and then passing the overriden PasswordResetConfirmView the user.pk that it would have normally received from the url.

This method hides the user.pk from the user but doesn't change any of the underlying functionality of the Django source.

#### CustomPasswordResetConfirmView.py

```python
from django.contrib.auth.views import PasswordResetConfirmView
from django.utils.encoding import force_bytes
from django.utils.http import urlsafe_base64_encode

# Change to match location of your models file
from app.models import Profile


class CustomPasswordResetConfirmView(PasswordResetConfirmView):
    def dispatch(self, *args, **kwargs):
        try:
            profile = Profile.objects.get(account_id=kwargs['uidb64'])
            kwargs['uidb64'] = urlsafe_base64_encode(force_bytes(profile.user.pk))
            return super(CustomPasswordResetConfirmView, self).dispatch(*args, **kwargs)
        except:
            return super(CustomPasswordResetConfirmView, self).dispatch(*args, **kwargs)
```

The only real difference needed in urls.py is to add our CustomPasswordResetConfirmView as the view for the url instead of the built in PasswordResetConfirmView.
I have provided the full content of my urls.py file just for context.

#### urls.py

```python
from django.conf.urls import url
from django.urls import reverse_lazy

from django.contrib.auth.views import (
    PasswordResetView,
    PasswordResetDoneView,
    PasswordResetCompleteView
)
# Change to point to real location
from views import CustomPasswordResetConfirmView

urlpatterns = [
    
    # Forgotten Password Page
    url(r'^accounts/password_reset/$', PasswordResetView.as_view(template_name='password_reset_form.html', email_template_name='account_templates/password_reset_email.html'), name='password_reset'),

    # Page shown after a user has been emailed a password reset link
    url(r'^accounts/password_reset/done/$', PasswordResetDoneView.as_view(template_name='password_reset_done.html'), name='password_reset_done'),

    # Form shown for entering a new password
    url(r'^accounts/reset/(?P<uidb64>[0-9A-Za-z_\-]+)/(?P<token>[0-9A-Za-z]{1,13}-[0-9A-Za-z]{1,20})/$',
        CustomPasswordResetConfirmView.as_view(
            post_reset_login=False,
            template_name='password_reset_confirm.html'
        ),
        name='password_reset_confirm'
    ),

    # Page showing user that password has been changed successfully
    url(r'^accounts/reset/done/$', PasswordResetCompleteView.as_view(template_name='password_reset_complete.html'), name='password_reset_complete'),

]
```

The final piece of the puzzle is to change the email sent to the user to reset their password.
The only difference between the below file and Django's example file is that I have replaced 'uidb64=user.pk' with 'uidb64=user.profile.account_id'.

#### account_templates/password_reset_email.html

```
Someone asked for password reset for email {{ email }}. Follow the link below:
{ { protocol } }://{ { domain } }{ % url 'password_reset_confirm' uidb64=user.profile.account_id token=token % }
```
