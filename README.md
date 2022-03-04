# InvalidAuthenticityToken and Rails CSRF

I receive exception notification for `InvalidAuthenticityToken` usually for
login or registration page. You can remove this exceptions if you extend session
time beyond the browser window (when user closes the browser).

This README is located
https://github.com/duleorlovic/invalid_authenticity_token_csrf/README.md

This example is based on
https://stackoverflow.com/questions/20875591/actioncontrollerinvalidauthenticitytoken-in-registrationscontrollercreate/60394170#60394170
and https://github.com/rails/rails/issues/21948

Start by creating new application and scaffold for Test model
```
rails new invalid_authenticity_token_csrf
cd invalid_authenticity_token_csrf
rails g scaffold Test test:string
rails db:create
rails db:migrate
rails s
```
and open http://localhost:3000/tests/new

Rails uses CSRF protection by inserting CSRF token into the form as hidden
field. You can find it in rails logs as `authenticity_token` parameter

```
Started POST "/tests" for ::1 at 2022-03-04 11:10:46 +0100
Processing by TestsController#create as TURBO_STREAM
  Parameters: {"authenticity_token"=>"[FILTERED]", "test"=>{"test"=>""}, "commit"=>"Create Test"}
  TRANSACTION (0.1ms)  BEGIN
  ↳ app/controllers/tests_controller.rb:27:in `block in create'
  Test Create (4.0ms)  INSERT INTO "tests" ("test", "created_at", "updated_at") VALUES ($1, $2, $3) RETURNING "id"  [["test", ""], ["created_at", "2022-03-04 10:03:10.370406"], ["updated_at", "2022-03-04 10:03:10.370406"]]
  ↳ app/controllers/tests_controller.rb:27:in `block in create'
  TRANSACTION (0.9ms)  COMMIT
  ↳ app/controllers/tests_controller.rb:27:in `block in create'
Redirected to http://localhost:3000/tests/1
Completed 302 Found in 16ms (ActiveRecord: 8.2ms | Allocations: 11496)
```

By default session cookie expires when user closes the browser. You can find
that when you inspect the page

![session cookie initial](https://github.com/duleorlovic/invalid_authenticity_token_csrf/raw/main/doc/session_cookie_initial.png "Session cookie")


Safari browser implements caching when we load pages from last session. Go to
'Safari' > 'Preferences...' > 'General' and set 'Safari opens with:' to 'All
windows from last session'.
* Now try to load the page with a form http://localhost:3000/tests/new
* Close safari with CMD+Q.
* Open Safari and you should see the form.
* When you submit the form you should see an error
```
Started POST "/tests" for ::1 at 2022-03-04 11:10:46 +0100
Processing by TestsController#create as TURBO_STREAM
  Parameters: {"authenticity_token"=>"[FILTERED]", "test"=>{"test"=>""}, "commit"=>"Create Test"}
Can't verify CSRF token authenticity.
Completed 422 Unprocessable Entity in 1ms (Allocations: 665)

ActionController::InvalidAuthenticityToken (Can't verify CSRF token authenticity.):
```

The same `InvalidAuthenticityToken` error occurs in iPhone simulator app (open
new page, close Safari by swipping up the app, start again Safari and submit the
form).

Note that I can not reproduce `InvalidAuthenticityToken` when I navigate to new
page by clicking on link *New test* on http://localhost:3000/tests (probably
since Rails 7 uses Turbo to navigate) since in this case when I reopen the
Sarafi I see new request
```
Started GET "/tests/new" for ::1 at 2022-03-04 11:29:11 +0100
```
which means that no cache is used. So if you use links to navigate the the form,
please refresh the page before closing the browser.

One simple solution is to extend session to let's say 1.month.
(find the cookie name inside Developer Tools -> Storage -> Cookies)
```
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store, key: '_invalid_authenticity_token_csrf_session', expire_after: 1.month
```

Restart the server and try again to open the form, close and open Safari. You
can notice that cache is still used (no request when you reopen Safari) but you
can submit the form without any errors.

Please note that if you use Devise, it will keep users logged in like they
clicked on *Remeber me* checkbox, so it is advisable to remove `:rememberable`.
One way to mitigate this is to use timeout feature, for example you can log out
user after 1 week of inactivity (admins after 1.day)
```
# app/models/user.rb
class User < ApplicationRecord
  # Include default devise modules. Others available are:
  # :confirmable, :lockable, :omniauthable, :rememberable,
  devise :database_authenticatable, :timeoutable, :trackable,
         :recoverable, :validatable, :confirmable

  def timeout_in
    if superadmin? || support?
      1.day
    else
      1.week
    end
  end
end
```
