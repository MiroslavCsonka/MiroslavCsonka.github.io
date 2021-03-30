---
title: Defining nested Ruby constants
classes: wide
tags:
  - Ruby
---

## Basics

Just so that we have something to build on, let's start with normal constants in Ruby.

They are usually defined in upper case with underscores.

```ruby

class Account
  STATUS_ACTIVE = 'active'
  STATUS_BANNED = 'banned'
end
```

## Problems with getting all constants

The most common way of having all of them available, is to just define it, like this:

```ruby

class Account
  STATUS_ACTIVE = 'active'
  STATUS_BANNED = 'banned'

  ALL_STATUSES = [STATUS_ACTIVE, STATUS_BANNED]
end
```

In th past, I needed it for:

* generating API documentation and showing all statuses `Account::ALL_STATUSES.to_sentence => "active and banned"`
* generating `<select>` HTML element
* validation `validates :status, inclusion: { in: ALL_STATUSES }` ([more in docs](https://guides.rubyonrails.org/active_record_validations.html#inclusion))

The main downside of this approach is that you have to remember to include it in `ALL_STATUSES`. For simpler cases
is not a problem, but it gets hairy for more complicated cases, such as:

```ruby

class Request
  SUCCESS = 200
  CREATED = 201
  CLIENT_ERROR = 400
  NOT_FOUND = 404
  SERVER_ERROR = 500

  SUCCESS_CODES = [SUCCESS, CREATED]
  CLIENT_ERROR_CODES = [CLIENT_ERROR, NOT_FOUND]
  SERVER_ERRORS = [SERVER_ERROR]

  # More common approach
  ALL_CODES = [SUCCESS, CREATED, CLIENT_ERROR, NOT_FOUND, SERVER_ERROR]

  # Little bit smarter approach
  ALL_CODES = SUCCESS_CODES + CLIENT_ERROR_CODES + SERVER_ERRORS
end
```

Now when you add a new code, you'll need to update the right success/client errors/server errors hash and potentially
even `ALL_CODES`.

> DISCLAIMER: Please do not roll you own HTTP client, this is just an example. In my previous jobs we dealt with payment statuses, loan application statuses, fraud statuses where it isn't that obvious how they relate to each other. Hence, we were more likely to make and a mistake, and unfortunately we made it a few times.

## The trick

The core of the "trick" is to understand return value of the constant definition is just the value you assigned:

```ruby
2.7.2 : 001 > CONSTANT = 1
=> 1
```

This allows us to write the `Request` like this:

```ruby

class Request
  ALL_CODES = [
    SUCCESS_CODES = [
      SUCCESS = 200,
      CREATED = 201,
    ],
    CLIENT_ERROR_CODES = [
      CLIENT_ERROR = 400,
      NOT_FOUND = 404,
    ],
    SERVER_ERRORS = [
      SERVER_ERROR = 500
    ],
  ].flatten
end
```

## Nicer git diff

When introducing new constant, with the smarter approach, the git diff is super obvious:

```diff

class Request
  ALL_CODES = [
    SUCCESS_CODES = [
      SUCCESS = 200,
      CREATED = 201,
    ],
    CLIENT_ERROR_CODES = [
      CLIENT_ERROR = 400,
+     FORBIDDEN = 403,
      NOT_FOUND = 404,
    ],
    SERVER_ERRORS = [
      SERVER_ERROR = 500
    ],
  ].flatten
end
```

Compared to "flattened" approach you have to update multiple places and as a reviewer, it is a bit trickier to know
what's changing

```diff

class Request
  SUCCESS = 200
  CREATED = 201
  CLIENT_ERROR = 400
+ FORBIDDEN = 403
  NOT_FOUND = 404
  SERVER_ERROR = 500

  SUCCESS_CODES = [SUCCESS, CREATED]
- CLIENT_ERROR_CODES = [CLIENT_ERROR, NOT_FOUND]
+ CLIENT_ERROR_CODES = [CLIENT_ERROR, FORBIDDEN, NOT_FOUND]
  SERVER_ERRORS = [SERVER_ERROR]

  # More common approach
- ALL_CODES = [SUCCESS, CREATED, CLIENT_ERROR, NOT_FOUND, SERVER_ERROR]
+ ALL_CODES = [SUCCESS, CREATED, CLIENT_ERROR, FORBIDDEN, NOT_FOUND, SERVER_ERROR]

  # Little bit smarter approach
  ALL_CODES = SUCCESS_CODES + CLIENT_ERROR_CODES + SERVER_ERRORS
end
```

## Conclusion

This obvious, yet simple trick shows the structure via nesting, is less error prone, `git diff` is clearer, and uses just
plain Ruby. What's not to love?
