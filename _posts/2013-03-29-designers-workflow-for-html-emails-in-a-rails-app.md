---
title: A Designer's Workflow for HTML Emails in a Rails App
link: http://devblog.orgsync.com/designers-workflow-for-html-emails-in-a-rails-app/
layout: single
author: tylerlee
comments: true
post_name: designers-workflow-for-html-emails-in-a-rails-app
tags: design
---

You already know why HTML Emails suck, but here's a quick list:

  1. Testing is arduous
  2. You can't use divs (Hello tables!)
  3. You have to inline style everything (really everything)
  4. When you inline things, they become impossible to update

### Hear This: I Fixed it.

The emails that our users at OrgSync receive from our app were getting stale, and no one wanted the task of reconfiguring them because of how much a pain it is to work with HTML emails. We have 27 individual views that generate emails, but those views get populated with content from all over our application--so this isn't the easiest of projects to begin with. Add in the list of things that suck about HTML emails and you get a decent-sized headache.

It took a little bit of setup and testing but now I can say that I've solved HTML emails. I've made it easy to write, test, and deploy. It's almost fun! Okay, let's not go that far, but I do think that the workflow we have is easier, and that makes it less painful. We're going to be able to make our emails more useful to our users because they're easier to work with on our end--and that's what it's all about, right?

So I've got some tips for your next email project and how to get over some of these hurdles:

#### Tip 1: Use Rails Layouts

I'm not sure when ActionMailer layouts got added to Rails, but I know that it wasn't in the iteration of Rails we were using when our emails were first written. Every view was hand-coded, with the entire template from header to footer embedded around the actual content of the email. For some reason, there seem to be apps that still do it this way, which boggles my mind, and makes me feel better we weren't the only ones behind the times there.

That being said: make the switch if you haven't. Pull out your template code, place it in a proper layout file like you would for anything else in Rails, and it works in the same way. Now when I need to make that change to the background color, I only have to change it in one spot, not 27.

##### Sidenote: Suck It Up and Write Code Like It's 1998

I didn't solve div support in email clients. I'm sorry if you thought I did; just embrace tables. Give up on trying to get divs to work. Use tables for everything. Use them like you did in 1998 and your email will be drastically more consistent than if you try to make divs work for layout.

#### Tip 2: Use CSS the Way You Want To

Let's look at an old email in our app. I'm sure you've seen this code before.

{% highlight html %}
  <body style="padding: 0px;margin: 0px;font-family: arial, verdana, sans-serif;font-size: 12px;line-height: 1.5em;">
      <center>
        <table width="600" style="font-size: 12px;line-height: 1.5em;">
          <tr style="font-size: 12px;line-height: 1.5em;">
            <td style="font-size: 12px;line-height: 1.5em;">
              <div id="header" style="font-size: 12px;line-height: 1.5em;width: 600px;margin: 0 auto;">
                <img src="https://s3.amazonaws.com/newsletter_images/welcome_header.png">
              </div><br>
            </td>
          </tr>
        </table>
        <table width="450" style="font-size: 12px;line-height: 1.5em;">
          <tr style="font-size: 12px;line-height: 1.5em;">
            <td style="font-size: 12px;line-height: 1.5em;">
              <div id="wrapper" style="font-size: 12px;line-height: 1.5em;width: 450px;margin: 0 auto;">
{% endhighlight %}

Every line has custom styles written into the HTML. This is the ugliest thing I've ever seen. Seriously. It was impossible to upgrade the look and feel of our emails without manually going through and changing every single line to reflect any new style. You can understand why these emails didn't get changed for so long. But that's how the world worked until Premailer saved the day.

Premailer (Pre-Flight for email) has been packaged as a gem. It does the hard work for you. It lets us write CSS like we normally do, in a CSS file, and then it automatically gets inlined on the fly as the email is generated by our app. While the end-result email looks exactly the same, we don't have to trifle with that madness of code above. We also love the fact we can still write in SASS, so it fits even better into our workflow. You still have to put on your designer hat and know what CSS does and does not work in email clients--but, holy smokes, is this a time saver. [Here's a railscast on using the Premailer Gem](http://railscasts.com/episodes/312-sending-html-email?view=asciicast)

  * Note: You can't inline media queries, that just doesn't work, but we still wanted responsive emails. Our setup: We have a SASS file that includes all of our base CSS for our emails. Everything in that file will get inlined into the HTML that the email generates. We include that into the layout via a link tag. In that same layout, we've also included a style block with all of our media query specifics. We've told preliner to **_not_** inline this block. Email clients that support media queries will still use the declarations in this block; those that don't allow media queries (or style blocks) will ignore them.

After this step is made and premailer is implemented, you're going to find new techniques you can use in your emails and you'll be able to generate markup differently because you won't have to worry about inlining those styles. I owe the premailer people a beer.

### Tip 3: Make Testing A Simple Process

The main reason designers hate designing emails is uncertainty. We are sometimes almost guessing as to the outcome of our code on the client side. With the number of emails OrgSync sends out, we had to make testing easier, and it had remove that uncertainty.

#### Testing functionality - Build a Rake Task

The first thing I want to test is that these emails work; that the data in them is being pulled correctly, that it's making it out of our server and into the real world, and that the basic design looks kosher.

How do you send a test email in your app now? You probably jump into the Rails console and do something like this:

{% highlight ruby %}
  MessageMailer.payment_message(account, organization, amount, payment_type, order_number).deliver
{% endhighlight %}

So we're sending a message to a user about the payment they made. Not so bad, it's only one line. Then you realize you have to build all of those objects first in order to pass them to your mailer. Depending on the types of objects we're dealing with, this could be a giant pain. But this is how OrgSync tested emails; if I needed to test this payment_message, I would jump in and spend time digging into the code, figuring out what this method was expecting, and finally come up with a working payment_message I could send. Then I would exit the console, come back tomorrow and do it all over again. This is dumb. It's even dumber when you multiply that by 27 emails.

Eventually I built myself a text file where I held a lot of these code snippets that set these messages up for me. Also dumb. I was the caretaker of the testing setup; it wasn't in the repository, so no one else knew the information that I had built up.

So I combined those snippets into a simple rake task. It's little more than a task that iterates over each snippet individually and provides a little bit of helpful output. It's a simple hack, but now anyone in the company can test on their own in a safe way.

Here's the setup for that task:

{% highlight ruby %}
  task :send_all => :environment do |acc|

    # Establish what user we want to send the test to
    print "\n What is your account id? "
    acc_num = $stdin.gets.to_i
    account = Account.find(acc_num)

    # This is a good spot to do a little input-error checking. I don't trust myself (or my coworkers)
    # so I ran a check to make sure the account here exists as an administrator in our system
    # this prevents me from sending our suite of test emails to a random user.
{% endhighlight %}