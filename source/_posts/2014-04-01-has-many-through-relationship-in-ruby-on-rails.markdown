---
layout: post
title: "Has many through relationship in Ruby on Rails"
date: 2014-04-01 21:45
comments: true
categories: RoR
---

Recently for some specific reasons I'm studying Ruby on Rails (RoR). One day I was stuck in a problem about how to handle many-to-many relationship.

Let's firstly see my example:

Suppose there're two models: `User` and `Skill`, a user can have many skills, and a skill may belong to many users. what's more, the proficiency of skills owned by different users may be different. For example, user-1 may be primary in skill_1 and normal in skill_2, while user-2 may be professional in skill_1, and primary in skill_2. 

So how to handle this kind of relationship in Ruby on Rails? And what if I want to add skill_3 to user-1, who are normal in such skill, then what should I do?

After searching google, I found two very useful tutorials: [one from RailsCast](http://railscasts.com/episodes/47-two-many-to-many), and [one from StackOverflow and Github](https://github.com/szines/hodor_filmdb).

For the first one from RailsCast, it is a tutorial about two ways to have many-to-many relationship: 

* has_and_belongs_to_many
* has_many :through

As said in the cast, `has_many :through` can be used when the joint table has some other fields.

So in my case, I should use `has_many :through` way, because in my joint table there may be a `proficiency` field.

For the second one, it is a example written by szines in order to reply to [a question on StackOverflow](), the example is about how to use `has_many :through` to realize many-to-many relationship.

According to the above two materials, I desided to use the `has_many :through` relationship. Then following is how I did that.

Firstly I add another Model called `Ability`, which has a user_id, skill_id and a field called "proficiency". The basic relationship looks like this:

![basic relationship](http://ytliu.info/images/2014-04-01-1.png "basic relationship")

In Ruby on Rails, for `User` model:

{% codeblock lang:ruby %}
class User < ActiveRecord::Base
  has_many :abilities
  has_many :skills, :through => :abilities

  ...

end
{% endcodeblock %}

for `Skill` model:

{% codeblock lang:ruby %}
class Skill < ActiveRecord::Base
  has_many :abilities
  has_many :users, :through => :abilities

  ...
  
end
{% endcodeblock %}

for `Ability` model:

{% codeblock lang:ruby %}
class Ability < ActiveRecord::Base
  belongs_to :user
  belongs_to :skill

  ...

end
{% endcodeblock %}

in `db/migrate/xxx_create_users.rb`

{% codeblock lang:ruby %}
class CreateUsers < ActiveRecord::Migration
  def change
    create_table :users do |t|
      t.string :name

      t.timestamps
    end
  end
end
{% endcodeblock %}

in `db/migrate/xxx_create_skills.rb`

{% codeblock lang:ruby %}
class CreateSkills < ActiveRecord::Migration
  def change
    create_table :skills do |t|
      t.string :name

      t.timestamps
    end
  end
end
{% endcodeblock %}

in `db/migrate/xxx_create_abilities.rb`

{% codeblock lang:ruby %}
class CreateAbilities < ActiveRecord::Migration
  def change
    create_table :abilities do |t|
      t.belongs_to :user, index: true
      t.belongs_to :skill, index: true
      t.string :proficiency

      t.timestamps
    end
  end
end
{% endcodeblock %}

So the final relationship looks like this:

![final relationship](http://ytliu.info/images/2014-04-01-2.png "final relationship")

Then, when we need to add a normal skill_3 to user-1, we can add code in users_controller.rb

{% codeblock lang:ruby %}
...

@user = User.find(1)
skill = Skill.where("name = skill_3").first
@user.abilities.create(proficiency: "normal", skill_id: skill.id)

...
{% endcodeblock %}

Here we done!