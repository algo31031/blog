---
layout: post
title: "Rails遗留程序里最常犯的错误(译)"
date: 2014-07-23 13:23:08 +0800
comments: true
categories: [rails, translation]
---

原文出自[most-common-mistakes-on-legacy-rails-apps](http://edelpero.svbtle.com/most-common-mistakes-on-legacy-rails-apps), 感谢作者[EZEQUIEL DELPERO](http://edelpero.svbtle.com/)

近来我一直在对几个遗留项目作维护。

众所周知，处理遗留项目多数时间都感觉糟透了，因为那些代码通常都丑陋不堪而且晦涩难懂。

我决定做一个列表，记录下那些公认的不良实践，或者是我认为不太好的实践，以及如何改良代码来避免这些不良实践。

### 问题一览

* 在模型层以外使用查询方法
* 在视图层使用业务逻辑
* 使用无意义的方法名和变量名
* 条件判断时使用unless或者否定的表达式
* 没有遵循“命令，不要去询问”原则
* 使用复杂的条件
* 在模型的实例方法里，本来不需要的时候使用了“self.”
* 使用条件表达式并且返回了条件
* 在视图层使用行内样式
* 在视图层使用JavaScript
* 调用方法时把另一个方法的调用作为参数
* 没有使用类来隔离Rake Tasks
* 没有遵循Sandi Metz的规则

###在模型层以外使用查询方法

**不好的**

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def index
    @users = User.where(active: true).order(:last_login_at)
  end

end
```

这段代码不可重用而且难于测试。如果你在别的地方也想查找全部用户并进行排序，就会出现冗余代码。

**好的**

比起在控制器里使用查询方法，我们的做法是在模型层中使用scope把它们独立出来，就如下例所示。这样做既可以使代码能够复用，又便于代码阅读和测试。

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  scope :active, -> { where(active: true) }
  scope :by_last_login_at, -> { order(:lasst_login_at) }

end
```

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def index
    @users = User.active.by_last_login_at
  end

end
```

每当你想用where、order、joins、includes、group、having或者其他查询方法时，记得要把它们放在模型层里。

###在视图层使用业务逻辑

**不好的**

```ruby
<!-- app/views/posts/show.html.erb -->
...
<h2>
  <%= "#{@comments.count} Comment#{@comments.count == 1 ? '' : 's'}" %>
</h2>
...
```

初看之下这小段代码似乎没什么问题，但是它会让HTML变得有点难以阅读，而且可以肯定的说一旦你在视图层添加了逻辑代码，那么日后你定会添加更多的逻辑到视图。这段代码还有一个问题，里面的逻辑无法复用，而且不能单独测试。

**好的**

使用Rails的helper方法把业务逻辑隔离出来

```ruby
# app/helpers/comments_helper.rb
module CommentsHelper
  def comments_count(comments)
    "#{comments.count} Comment#{comments.count == 1 ? '' : 's'}"
  end
end
```

```ruby
<!-- app/views/posts/show.html.erb -->
<h2>
  <%= comments_count(@comments) %>
</h2>
```

###使用无意义的方法名和变量名

**不好的**

```ruby
# app/models/topic.rb
class Topic < ActiveRecord::Base

  def self.r_topics(questions)
    rt = []

    questions.each do |q|
      topics = q.topics

      topics.each do |t|
        if t.enabled?
          rt << t
        end
      end
    end

    Topic.where(id: rt)
  end

end
```

这类遗留代码最主要的问题在于：你需要花费大把时间来搞清楚这些代码的用途。r_topics这个方法是做什么的，rt这个变量又是什么意思。其他的一些变量，比如在代码块里用到的那个，变量名的含义很模糊，这样也使得它们的用途初看起来很难理解。

**好的**

对方法和变量命名时选那些能表达出其含义的名字。这样更便于其他开发者理解你的代码。

```ruby
# app/models/topic.rb
class Topic < ActiveRecord::Base

  def self.related(questions)
    related_topics = []

    questions.each do |question|
      topics = question.topics

      topics.each do |topic|
        if topic.enabled?
          related_topics << topic
        end
      end
    end

    Topic.where(id: related_topics)
  end

end
```

这样改进的好处在于：

* 第一次看到方法名时就会对方法返回值有个概念。一个与给定问题集合相关联的主题的集合。
* 现在你能够了解related_topics表示一个数组，它里面存放了一个与给定问题集合相关联的主题的集合。之前打代码里rt表示什么非常含糊。
* 使用topic代替之前的t，并用question替换掉q，使得你的代码更便于阅读，因为你不再需要脑补这些变量的用途。现在这些代码已然能够自解释一切。

###条件判断时使用unless或者否定的表达式

**不好的**

```ruby
# app/services/charge_user.rb
class ChargeUser

  def self.perform(user, amount)
    return false unless user.enabled?

    PaymentGateway.charge(user.account_id, amount)
  end

end
```

这段代码也许并不难理解，但是使用unless或者否定的条件表达式会稍微增加代码的发复杂度，因为你必须对它要判断的条件自行脑补。

**好的**

改用if或者肯定的条件表达式之后，上述代码就会好懂得多。

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def disabled?
    !enabled?
  end

end
```

```ruby
# app/services/charge_user.rb
class ChargeUser

  def self.perform(user, amount)
    return false if user.disabled?

    PaymentGateway.charge(user.account_id, amount)
  end

end
```

不觉得这样写代码更易读了吗？我更倾向于使用if而非unless，用肯定的表达式多过肯定的表达式。实在不行就添加个反意的方法，比如我们在User模型里加的那个。

###没有遵循“命令，不要去询问”原则

**不好的**

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def enable!
    update(enabled: true)
  end

end
```

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def enable
    user = User.find(params[:id])

    if user.disabled?
      user.enable!
      message = "User enabled"
    else
      message = "User already disabled"
    end

    redirect_to user_path(user), notice: message
  end

end
```

这里的问题是在不恰当的地方出现了控制逻辑。你先判断了用户是否是不可用，如果的确不可用，就启用这个用户。

**好的**

比较好的改办法是把控制逻辑放到enable!方法里。

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def enable!
    if disabled?
      update(enabled: true)
    end
  end

end
```

```ruby
# app/controllers/users_controller.rb
class UsersController < ApplicationController

  def enable
    user = User.find(params[:id])

    if user.enable!
      message = "User enabled"
    else
      message = "User already disabled"
    end

    redirect_to user_path(user), notice: message
  end

end
```

现在控制器不用关心user需要满足何种条件才会被启用。相关的判断由User模型和enable！方法来处理。

###使用复杂的条件

**不好的**

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController

  def destroy
    post = Post.find(params[:id])

    if post.enabled? && (user.own_post?(post) || user.admin?)
      post.destroy
      message = "Post destroyed."
    else
      message = "You're not allow to destroy this post."
    end

    redirect_to posts_path, notice: message
  end

end
```

条件表达式弄的太过复杂了，实际上这里只想知道一件事：用户可以删掉post吗？

**好的**

从上面的代码我们可以了解到，一个用户需要是post的所有者或者这个用户是管理员，并且post本身也是可用的，才可以删除这个post。最好的做法就是，把这些条件抽取成一个日后可以复用的方法。

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def can_destroy_post?(post)
    post.enabled? && (own_post?(post) || admin?)
  end

end
```

```ruby
# app/controllers/posts_controller.rb
class PostsController < ApplicationController

  def destroy
    post = Post.find(params[:id])

    if user.can_destroy_post?(post)
      post.destroy
      message = "Post destroyed."
    else
      message = "You're not allow to destroy this post."
    end

    redirect_to posts_path, notice: message
  end

end
```

每当条件表达式里出现了&&或者||，就应该把它们提取为方法，以备以后复用。

###在模型的实例方法里，本来不需要的时候使用了“self.”

**不好的**

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def full_name
    "#{self.first_name} #{self.last_name}"
  end

end
```

这段代码并不复杂但是里面并不需要使用“self.”。把“self.”去掉会使代码更简洁且不影响可用性。

**好的**

在模型里，只有在实例方法里需要赋值时，才会用到“self.”，否则通篇的“self.”只会徒增代码复杂度。

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def full_name
    "#{first_name} #{last_name}"
  end

end
```

###使用条件表达式并且返回了条件

**不好的**

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def full_name
    if name
      name
    else
      "No name"
    end
  end

end
```

或者

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def full_name
    name ? name : "No name"
  end

end
```

这段代码的问题在于：在不需要的地方添加了控制语句。

**好的**

有种更简便的处理方式也能达到同样效果

```ruby
# app/models/user.rb
class User < ActiveRecord::Base

  def full_name
    name || "No name"
  end

end
```

简单来说这段代码会在name不为false或nil时将其返回，否则返回"No name".

使用得当的话，||和&&这些操作符会对提升你的代码品质提供巨大助力。

###在视图层使用行内样式

**不好的**

```ruby
<!-- app/views/projects/show.html.erb -->
...
<h3 style="font-size:20px;letter-spacing:normal;color:#95d60a;line-height:100%;margin:0;font-family:'Proxima Nova';">
  SECRET PROJECT
</h3>
...
```

这里我们只列出一个标签，所有的样式都写在了标签里。现在，请设想一下，如果所有的标签都接收行内样式。这会把你的HTML变得和其难度，除此之外，每当你需要引入另一个同样的h3元素时，将不得不把同样代码照搬一边，造成冗余。

**好的**

```ruby
// app/assets/stylesheets/application.css
.project-title {
    font-size: 20px;
    letter-spacing: normal;
    color: #95d60a; 
    line-height: 100%;
    margin: 0;
    font-family:'Proxima Nova';
}
```

```ruby
<!-- app/views/projects/show.html.erb -->
...
<h3 class="project-title">
  SECRET PROJECT
</h3>
...
```

现在我么可以复用样式了，并且HTML的可读性也有所提高。

**注意：**这只是个简单的范例，实际应用时你应该把CSS拆分成多个小文件，并通过application.css来加载这些文件。另外只有在email模板里，才会用到行内样式。

###在视图层使用JavaScript

**不好的**

```ruby
<!-- app/views/questions/show.html.erb -->
...
<textarea rows="4" cols="50" class='wysihtml5'>
  Insert your question details here.
</textarea>
...
<script>
  $(document).ready(function(){
  $('textarea.wysihtml5').wysihtml5({
    "font-styles": true, //Font styling, e.g. h1, h2, etc. Default true.
    "emphasis": true, //Italics, bold, etc. Default true.
    "lists": true, //(Un)ordered lists, e.g. Bullets, Numbers. Default true.
    "html": false, //Button which allows you to edit the generated HTML. Default false.
    "link": true, //Button to insert a link. Default true.
    "image": true, //Button to insert an image. Default true.
    "color": true //Button to change color of font. Default true.
  });
});
<script>
```

这里的逻辑和特定页面耦合在一起，导致代码不可复用。

**好的**

Rails里面有专门用于组织和存放javascript代码的地方：“app/assets/javascripts/”。

```ruby
// app/assets/javascripts/application.js
...
$(document).ready(function(){
  $('textarea.wysihtml5').wysihtml5({
    "font-styles": true, //Font styling, e.g. h1, h2, etc. Default true.
    "emphasis": true, //Italics, bold, etc. Default true.
    "lists": true, //(Un)ordered lists, e.g. Bullets, Numbers. Default true.
    "html": false, //Button which allows you to edit the generated HTML. Default false.
    "link": true, //Button to insert a link. Default true.
    "image": true, //Button to insert an image. Default true.
    "color": true //Button to change color of font. Default true.
  });
});
...
```

```ruby
<!-- app/views/questions/show.html.erb -->
...
<textarea rows="4" cols="50" class='wysihtml5'>
  Insert your question details here.
</textarea>
...
```

现在我们可以在view层任何地方用这段代码了。只需要页面上有一个带有wysihtml5这个class的textarea，刚才的那段js就会被执行。

**注意：**这只是个简单的范例，实际应用时需要考虑是否需要把你的JavaScript拆分成若干小的文件，并通过application.js来加载这些文件。另外，如果你使用的是CoffeeScript而非JavaScript，请坚持不要把CoffeeScript与普通JavaScript在一起混写。

###调用方法时把另一个方法的调用作为参数

**不好的**

```ruby
# app/services/find_or_create_topic.rb
class FindOrCreateTopic
  ...
  def self.perform(user, name)
    find(user, sluggify(name)) || create(user, name)
  end
  ...
end
```

这段代码里调用了find方法并传入了2个参数，首参数为user，第二个参数则是直接调用了sluggify这个方法并把name作为参数传给sluggify。你也许会有疑问，这么写有什么问题吗？我明明完全能够看懂这段代码呀。是的，代码也许不难理解，但是每次到这里你都需要自己做一点脑筋转换，而这正是我一直极力想要避免的。

**好的**

避免需要脑筋转换的一个比较有效的办法就是：使用有意义的变量名。

```ruby
# app/services/find_or_create_topic.rb
class FindOrCreateTopic
  ...
  def self.perform(user, name)
    slug = sluggify(name)
    find(user, slug) || create(user, name)
  end
  ...
end
```

这样做可以避免脑筋转换。换用含义明确的变量名之后，每当你再调用find方法，就会知道find接受一个user和一个slug做参数。

###没有使用类来隔离Rake Tasks

**不好的**

```ruby
# lib/tasks/recalculate_badges_for_users.rake
namespace :users do

  desc "Recalculates Badges for Users"
  task recalculate_badges: :environment do
    user_count = User.count

    User.find_each do |user|
      puts "#{index}/#{user_count} Recalculating Badges for: #{user.email}"

      if user.questions_with_negative_votes >= 1 || user.answers_with_negative_votes >= 1
        user.grant_badge('Critic')
      end

      user.answers.find_each do |answer|
        question   = answer.question
        next unless answer && question

        days       = answer.created_at - question.created_at
        days_count = (days / 1.day).round

        if (days_count >= 60) && (question.vote_count >= 5)
          grant_badge('Necromancer') && return
        end
      end
    end
  end

end
```

这个rake task有问题多多。最主要的问题是，这个rake太长了而且不好测试。代码写的初一看也很难理解。你只好相信这个task的确是为用户重新计算奖章系统的，因为它上面描述就这么写的。

**好的**

解决这个问题最好的办法就是，把业务逻辑挪到一个特定的类里面。所以，让我们新建个类来搞定它吧。

```ruby
# app/services/recalculate_badge.rb
class RecalculateBadge
  attr_reader :user

  def initialize(user)
    @user = user
  end

  def grant_citric
    if grant_citric?
      user.grant_badge('Critic')
    end
  end

  def grant_necromancer
    user.answers.find_each do |answer|
      question = answer.question
      next unless answer && question

      if grant_necromancer?(answer, question)
        grant_badge('Necromancer') && return
      end
    end
  end

  protected

    def grant_citric?
      (user.questions_with_negative_votes >= 1) ||
      (user.answers_with_negative_votes >= 1)
    end

    def days_count(answer, question)
      days = answer.created_at - question.created_at
      (days / 1.day).round
    end

    def grant_necromancer?(answer, question)
      (days_count(answer,question) >= 60) &&
      (question.vote_count >= 5)
    end

end
```

```ruby
# lib/tasks/recalculate_badges_for_users.rake
namespace :users do

  desc "Recalculates Badges for Users"
  task recalculate_badges: :environment do
    user_count = User.count

    User.find_each do |user|
      puts "#{index}/#{user_count} Recalculating Badges for: #{user.email}"
      task = RecalculateBadge.new(user)
      task.grant_citric
      task.grant_necromancer
    end
  end

end
```

如你所见，现在这个rake task可读性要好的多，而且还可以单独测试每一种批准徽章的方法。除此以外我么也可以在有需要时复用这个类。当然这段代码只是点到即止，诸位可以再做进一步优化。

###没有遵循Sandi Metz的规则

1. 每个类代码不可以超过100行
2. 每个方法代码不可以超过5行
3. 方法参数不可以超过4个，hash项也包括在内
4. 控制器之可以初始化一个对象。而且视图层只可以使用一个实例变量，并且只可以在这个对象上调用方法（@object.collaborator.value这种是不可以的）。

更多关于Sandi Metz的规则请移步至[thoughtbot](http://robots.thoughtbot.com/),参阅[Sandi Metz' Rules For Developers](http://robots.thoughtbot.com/sandi-metz-rules-for-developers)这篇博文。