---
layout: post
title: Final report for GSoC '20
subtitle: My journey with rubygems.org
gh-repo: vachhanihpavan/rubygems.org
gh-badge: [star, fork, follow]
cover-img: /assets/img/2020-08-16/cover.png
thumbnail-img: /assets/img/2020-08-16/thumb.png
tags: [blog, gsoc, ruby, rubygems.org]
---

## Background
`gem` is a standard format for distributing Ruby libraries or packages provided by RubyGems package manager.
[rubygems.org](https://rubygems.org) is the Ruby community's gem hosting service to publish and fetch gems.
The user of RubyGems who has permission who push/yank any version of a gem are called the *owners* of that gem.

## My Project
My Google Summer of Code project with Ruby was focused on "Adding functionality of add/remove gem owners via Web UI".
The aim of this project is ensure frictionless management of ownerships without setting up CLI. This project will also
help continue maintaining gems by pointing interested users to gems which need maintainers and reviving unmaintained ones.

This blog post is for my final submission of GSoC project. 
 

- Project: [Add gem owner add and remove in web UI of rubygems.org](https://summerofcode.withgoogle.com/projects/#5841025752367104)
- Mentor: [Aditya Prakash](https://github.com/sonalkr132)
- Organisation: [Ruby](https://summerofcode.withgoogle.com/organizations/6676829491953664/)
- Project Page: [https://rubygsoc.github.io/add-gem-owner-add-and-remove-in-web-ui-of-rubygems.org/](https://rubygsoc.github.io/add-gem-owner-add-and-remove-in-web-ui-of-rubygems.org/)

Under this project, there were 2 parts to be implemented:
1. Functionality to add/remove owners of a gem via web UI similar to [CLI](https://guides.rubygems.org/command-reference/#gem-owner)
2. New ownership transfer flow where owners can put up a gem for adoption and interested users can apply for ownership

Pull requests created in the period of GSoC:

| **PR Title**                                                                                              | **Review Status** | **Merge Status** |
|:---------------------------------------------------------------------------------------------------------:|:-----------------:|:----------------:|
| [Ownership Confirmation](https://github.com/rubygems/rubygems.org/pull/2357)                              | Approved          | Open             |
| [Ownership Transfer](https://github.com/vachhanihpavan/rubygems.org/pull/22)                              | Pending           | Open             |
| [RFC](https://github.com/rubygems/rfcs/pull/25)                                                           | Approved          | Open             |
| [Guides](https://github.com/rubygems/guides/pull/265)                                                     | Pending           | Open             |
| **Other PRs**                                                                                                                                    |
| [Show yanked gem in search filters](https://github.com/rubygems/rubygems.org/pull/2339)                   | Approved          | Merged           |
| [Fix SimpleCov for correct coverage report](https://github.com/rubygems/rubygems.org/pull/2438)           | Approved          | Merged           |
| [Removed unnecessary ES cookie from tests](https://github.com/rubygems/rubygems.org/pull/2442)            | Approved          | Merged           |
| [Move owners integration test to API directory](https://github.com/rubygems/rubygems.org/pull/2471)       | Approved          | Merged           |
| [Extracted mailer partial to make code DRY](https://github.com/rubygems/rubygems.org/pull/2421)           | Approved          | Merged           |
| [Added route to redirect /users to /profile](https://github.com/rubygems/rubygems.org/pull/2264)          | Approved          | Closed           |

*I'll update the status of the PRs as they get changed*

## Project Goal
When a rubygem has multiple maintainers, it is important to keep track of all the owners who have access to
push, yank or add more owners to the gem. Currently, `gem owner --add` and `gem owner --remove` commands are used
to modify the owners list for a gem. The goal of this project is to add features of adding/removing the owners
using Web UI of rubygems.org without the hassle of setting up CLI. Alongside, the functionality to notify the owners about changes
in the ownership of the gem they own is to be added to the API and web which will improve mitigation time of unintended access. 

The next part of the project is about the transfer of ownership. The goal of this feature is to provide the gem maintainers with
a platform within rubygems.org where they can look for new maintainers of the project. The owners of a gem can open an ownership call
visible to all the users of rubygems.org who can apply to the call. The owner can choose from all the applications
and the users will be added as owners.

## Work Done

### Part 1: Add/remove owners using Web UI
The first part of the project involved three major components:

#### Ownership Confirmation
Before adding features of adding or removing owners via Web UI, I had to modify the existing API endpoints `POST - /api/v1/gems/[GEM NAME]/owners`
to add email confirmation before a user is added as an owner. This modification was necessary to avoid adding a new owner
without any confirmation from the user being added as an owner as it can potentially lead to email notification spamming.
The confirmation step on owner addition, we will avoid sending unsolicited emails.

![Ownership Confirmation](/assets/img/2020-08-21/confirmation.png)

#### Ownership Management
Next part I worked on was to add controllers and views for listing, adding and removing owners of a gem. I added new routes as below:
```ruby
resources :owners, only: %i[index destroy create], param: :handle do
    get 'confirm/:token', to: 'owners#confirm', as: :confirm, on: :collection
    get 'resend_confirmation', to: 'owners#resend_confirmation', as: :resend_confirmation, on: :member
end
```

With suggestions from my mentor Aditya and other members of the community, I added extra information about the owners like MFA status along with 
data for auditing the ownership changes like owner added date and the user who added.

I worked on two different UI designs for the ownership page. After a lot of discussions, suggestions and changes, final UI looked as:

![Owners Index](/assets/img/2020-08-21/owners-index.png)

A special thanks to [Hiren](https://github.com/hmistry) for providing constructive feedback and suggestions.

#### User Verification
Aditya suggested adding password verification of users before performing any sensitive actions of adding and removing owners. This would help
mitigate any unauthorized access due to active browser sessions of logged in users.
With this feature, a logged in user will be asked to confirm the password before performing any actions related to owners of a gem.
After confirming the password, user won't be asked again for next 10 minutes.

### Part 2: Ownership Transfer
The second part of the project involved two major components:

#### Ownership Calls
There are a few cases when a gem needs new maintainers:
1. When a gem owner isn't able to cope and needs help in maintaining.
2. When a gem owner is not interested in maintaining the gem anymore.

An ownership call can be opened in any of such cases.
Users who are interested in taking the project further can apply and contact the gem owner easily.

For this feature, I added a new endpoint `/ownership_calls` for a site wide listing of ownership calls for all rubygems. And I added
`GET /gems/<gem-name>/ownership_calls` to view a call, `POST /gems/<gem-name>/ownership_calls` to create a call
and `PATCH /gems/<gem-name>/ownership_calls` to close a call.

![Ownership Calls](/assets/img/2020-08-21/ownership-calls.png)

#### Ownership Requests
Next was to implement functionality for users to apply to a call with a small message for the owner. After a user applies to a call,
the owner will be notified about the applications using a digest email. Then, the owner can accept any applications and close the rest.

I also implemented an endpoint `GET /profile/ownership_calls` to list all ownership calls and ownership applications made by the user
at one place.

## Work Left To Do
The review for ownership transfer PR is pending. Based on the review comments, I will have to make the changes if required to get the PR merged.

## Weekly Updates
I have posted my weekly updates on Ruby GSoC blog [here](http://rubygsoc.github.io/add-gem-owner-add-and-remove-in-web-ui-of-rubygems.org/).

## Learning
- The major learning outcome from this project was good coding practice. I learnt how to write DRY, reusable code following SOLID principles.
I also learnt how to write good tests/specs covering all possible scenarios.

- When writing the [RFC](https://github.com/rubygems/rfcs/pull/25/) for the project, I had long discussions with Aditya trying to decide on
the terms to use for `ownership calls` and `ownership requests`. Coming up with good names for methods, variables and tests was a great lesson
learnt.

- While working on UI/UX for owners page, I learnt a lot of good practices for design as suggested by Hiren, Aditya and from online resources.

- Through this journey, I learnt a lot about how open source projects work, getting involved with the community and contributing to the projects.

- Alongside the project, I was reading a few books suggested by my mentor Aditya. They helped a lot in learning how to good code which
improves readability, productivity and reduces the cost of change.
    - PRACTICAL OBJECT ORIENTED DESIGN IN RUBY by Sandi Metz.
    - REFACTORING by Martin Fowler.
    
## Special Thanks
It was wonderful working with Ruby community throughout the GSoC project. I would like to thank Aditya for patiently reviewing my code,
giving constructive suggestions and guiding me through the program. Thanks to Hiren and the members of Ruby Community for helping with their
suggestions.

It would like to thank [Mohit Tahiliani](https://github.com/mohittahiliani) sir and [Mishal Shah](https://github.com/mishal23) for motivating
and guiding me in applying and getting through the whole program.  

Final thanks to Google for organizing this amazing program. I feel GSoC made it easier for me to get started with open source contributions.
I saw ideas getting dropped after PR was complete([#2264]((https://github.com/rubygems/rubygems.org/pull/2264))),
complete change in implementation([#2339](https://github.com/rubygems/rubygems.org/pull/2339)) and work done by me being deployed through the program.
It was a great lesson on how projects are built and maintained with great quality.
I am excited to continue contributing to the open source community.
