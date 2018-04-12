---
layout: post
title:      "Rails Portfolio Project - Verifying Pets' Health"
date:       2018-04-12 01:49:08 +0000
permalink:  rails_portfolio_project_-_verifying_pets_health
---


With my Rails portfolio project looming and no real viable ideas to work with, I wound up having a lightbulb moment during a conversation with my girlfriend's mother during a recent visit to Michigan. She was talking about how unfortunate it was that they were unable to make their customary visit to Disneyland the last time she visited Los Angeles, where we live, because she'd brought her dog with her and didn't want to leave him alone in her daughter's apartment all day. I suggested that next time she should consider one of the many pet hotels in L.A. (only the most luxurious would suffice--she really dotes on this dog), but she said that it's usually difficult to set that up on short notice because of all of the verifications such services need, for things like vaccinations and other health issues, in order to prevent anything nefarious from being communicated between their patrons.

I thought a third-party service for cataloging this information, where anyone could look up an animal and see the status of a given vaccination or other health screening, would satisfy the requirements for my Rails project pretty well, so I set to work when I got home from Michigan. My idea was to set up a web site where pet owners could log in and register their pets, which would initiate with a set of common health screenings depending on the pet's species. Veterinarians could log in and update the status of these screenings, which would be visible to anyone who wanted to verify the healthiness of any pet in the database. I had fun setting up this domain because it pushed me into some new territory, and also required a lot more back-end programming than much of what I've done in Rails so far. 

Because I wanted to have two different types of users, an Owner class and a Veterinarian class that would both inherit from my User class, enabling me to establish different associations between User types and other models in my domain, I had to do a little bit of reading on [single table inheritance](http://eewang.github.io/blog/2013/03/12/how-and-when-to-use-single-table-inheritance-in-rails/), or STI (an unfortunate acronym, imho). It turned out to be pretty straightforward (at least for my purposes), but did lead to some hurdles on the front end, particularly when it came to using form helpers, but I'll get to that later. All I really had to do was include a `:type` attribute in my CreateUsers migration and ActiveRecord handled the rest! With STI, there was no need to create tables for my child classes; all instances are persisted to the `users` table with the type set automatically by the subclass name upon creation, while still allowing you to treat subclasses like any other ActiveRecord model. All of this allowed me to set up my associations such that Owners and Veterinarians were associated in a many-to-many relationship through Pets, and instance of which `has_many` Veterinarians and `belongs_to` an Owner. I created a HealthScreening resource bearing a belongs_to/has_many relationship to a Pet and was ready to move forward.

Once I had all of my tables migrated and associations set up, I got to work on my models. I set up a lot of the predictable validations, like the following for my Pet class:

```
  validates :name, presence: true
  validates :species, presence: true, inclusion: {in: %w(Dog Cat)}
  validates :owner_id, presence: true
  validates :birth_date, presence: true
  validates :sex, presence: true, inclusion: {in: %w(Male Female)}
```

I didn't want anyone to be able to put a `last_updated` date on a screening that was prior to the pet's date of birth, so I set up a custom validation to handle that:

```
  validate :last_updated_cannot_be_prior_to_birth_date
	
  def last_updated_cannot_be_prior_to_birth_date
    if last_updated && last_updated.year*12 + last_updated.month < pet.birth_date.year*12 + pet.birth_date.month
      errors.add(:last_updated, "Cannot be prior to pet's date of birth.")
    end
  end
```

I also set up some ActiveRecord callbacks for my HealthScreening and Pet classes to handle some life-cycle events for me. Because a HealthScreening can go out of date or become current again based on the date it was last updated, I knew I'd need a method to check its status and update it if needed any time the database was queried for the record:

```
  after_find :update_status
	
  def update_status
    if months_since_last_updated
      if months_since_last_updated > renewal_interval*12
        self.status = "Overdue"
      else
        self.status = "Current"
      end
    else self.status = "Overdue"
    end
    self.save
  end
```

Because I wanted a Pet to instantiate with a suite of species-specific HealthScreenings that could be updated by a Veterinarian, I wrote a method and callback to handle that as well:

```
  after_initialize :initialize_screenings

  def initialize_screenings
    if self.health_screenings.empty?
      (self.cat? ? HealthScreening.cat_screenings : HealthScreening.dog_screenings).each do |name, interval|
        self.health_screenings.build(kind: name, renewal_interval: interval, species: self.species)
      end
    end  
  end
```

The issue I originally ran into with `after_initialize` was that it actually calls the method any time the database is queried for the record, so it kept initializing new HealthScreenings every time I viewed a Pet's status page, which was the reason for the conditional in `initialize_screenings`. This method also created another issue on the front end that took me hours to figure out. I'd initially had a call to save the Pet record after closing the `if` statement, but by the time I was writing the form to register a pet, that brief line of code was nowhere near the forefront of my mind. However it turned out to be the reason the form loaded with a whole set of error messages even before one tried to register a pet. Lesson learned. 

After writing some more helpful methods to my models, like these scoping methods for HealthScreening,

```
  def self.current
    where(status: "Current")
  end

  def self.overdue
    where(status: "Overdue")
  end
```

I moved on to routes, controllers, and views. 

I knew I'd be setting up a bunch of nested routes like `/users/:user_id/pets/new` and `/pets/:id/health_screenings/:status`, but I wasn't to sure of everything I'd need at the onset, so I created almost everything I could possibly need and then pared it down to the ones that were actually in use once I was done creating all of my actions and views. Between custom routes with helpers and nesting, this turned out to be a great lesson in Rails routing. I also created some helper methods right away that I knew I'd be needing down the line:

```
def logged_in?
      session[:user_id] ? true : false
    end

    def vets_only
      if current_user.class.name != "Veterinarian"
        flash[:alert] = "Sorry, that action is only available to veterinarians."
        redirect_to user_path(current_user)
      end
    end

    def owners_only
      if current_user.class.name != "Owner"
        flash[:alert] = "Sorry, that action is only available to pet owners."
        redirect_to user_path(current_user)
      end
    end

    def logged_in_only
      if !logged_in?
        flash[:alert] = "Please log in to view that page."
        redirect_to login_path
      end
    end
```

I got started on sessions first, which was where the first real consequences of my STI implementation set in. I'd been considering using Devise to set all of that up automatically, but realized that wouldn't accommodate my subclasses, so I went custom. My attempts to use a `form_for` helper to create new users were in vain, because `form_for` expects the argument passed to it to be a User, so it became easier to make users with a `form_tag`. I also needed a way to create a user as either an Owner or a Veterinarian, so I added a check box to the form and the following logic to my `users#create` action:

```
  def create
    if params[:vet]
      @user = Veterinarian.new(user_params)
    else
      @user = Owner.new(user_params)
    end

    if @user.save
      session[:user_id] = @user.id
      redirect_to user_path(@user)
    else
      render :new
    end
  end
```

Most of the rest of my work was pretty straightforward Rails programming. I set up a lot of dynamic content in the views, and some helpers to clean them up, particularly to tailor them based on what type of user was logged in. I added some authorization helpers to my controller actions to keep Owners from updating HealthScreenings and prevent Veterinarians from registering pets, which I saw as an Owner's responsibility. I created a helper to make the navigation links in my header specific to whether a user was logged in, a vet, or an owner: 

```
  def nav_header
    nav_html = content_tag(:li, link_to("Home", root_path))
    if logged_in?
      nav_html += content_tag(:li, link_to("Sign Out", logout_path))
      if current_user.owner?
        nav_html += content_tag(:li, link_to("My Pets", user_pets_path(current_user))) +
                    content_tag(:li, link_to("Register a New Pet", new_user_pet_path(current_user)))
      elsif current_user.vet?
        nav_html += content_tag(:li, link_to("My Patients", user_pets_path(current_user)))
      end
    else
      nav_html += content_tag(:li, link_to("Sign Up", new_user_path)) +
                  content_tag(:li, link_to("Sign In", login_path))
                  content_tag(:li, link_to("Find a Pet", pets_path))
    end
    nav_html += content_tag(:li, link_to("Find a Pet", pets_path))
  end
```

I also DRYed up my views with partials wherever I could, and did my best to outsource any logic in the views to helpers when it got a little too lengthy. 

One thing I did to venture beyond the requirements of the project was set up a search option on the pet index page, where users could look up a pet using their name, species, and owner's name. I thought it was a nice, and also appropriate, touch for an app that is supposed to help certain users quickly determine the status of a pet's health. Check out my [github repo](https://github.com/tpetersen0308/my-pet-health_app) for the app if you'd like to see it in a little more detail. 


