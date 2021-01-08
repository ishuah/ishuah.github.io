---
layout: post
title: Preact-Rails Gem&#58; Preact integration for Ruby on Rails.
description: We\'re happy to announce a new open source gem, preact-rails!
date: 2019-09-24 00:00:00
image: /images/22.jpg
tags: [developer-tools]
---

Our vision at Hover is to empower developers with resources, services, and tools to build for local communities at global scale. As part of this vision, we’re happy to announce a new open source gem, [preact-rails](https://github.com/UseHover/preact-rails)!

A few weeks ago we wrote about how our [developers outgrew our dashboard](https://medium.com/use-hover/our-developers-outgrew-our-mvp-83221f83bb77) within two months of our official launch. We took this as an opportunity to improve our dashboard UI and UX.

The first step was picking a javascript UI library. We had a lot of options but the ones that stood out were Vue.js and React. Part of the engineering team was familiar with React and it proved robust enough so we went ahead and picked that. You’re probably wondering why I’m mentioning React when the title clearly states “Preact”. Keep reading, it gets better.

Our dashboard is built on Ruby on Rails which is, by all accounts, very opinionated. Decoupling the front-end to an independent project would cause a lot of integration pains, the biggest being managing user sessions without compromising security. We wanted to upgrade the front-end without committing major changes to the back-end engine. The driving factor here was time, we didn’t have enough of it (do we ever?).

Taking this into consideration, we went back to our javascript UI library choice, React. We chose react before we made the choice not to build a whole front-end project on it’s own. The most feasible option was to serve react components on top of Rails views. It’s minimal and it serves our purpose of not intruding on the back-end and there was a gem just for this, [react-rails](https://github.com/reactjs/react-rails).

We started testing the gem by building a few simple components: buttons and forms. Around this time [David Kutalek](https://medium.com/u/50dd4d42d4e9) came across [Preact](https://preactjs.com/) (mentioned to him by our head of design, [Justin Scherer](https://medium.com/u/fe7d40e3c821)). We were intrigued. In a nutshell, Preact is a fast 3kB alternative to React. There are subtle differences that are addressed by a compatibility layer [preact-compat](https://github.com/developit/preact-compat). We wanted to test it out before we jumped in head first. There was one issue: we couldn’t find a gem similar to [react-rails](https://github.com/reactjs/react-rails), a Preact implementation for Ruby on Rails. That’s when [preact-rails](https://github.com/UseHover/preact-rails) was conceptualized. Inspired (and heavily influenced) by [react-rails](https://github.com/reactjs/react-rails), we started building a Preact implementation for Ruby on Rails.

This is where we get a tiny bit technical. There are two parts of this project. The first part is the [ruby gem](https://rubygems.org/gems/preact-rails), aptly named preact-rails. The gem serves one purpose, to make [preact\_component](https://github.com/UseHover/preact-rails/blob/master/lib/preact/view_helper.rb#L2) available as a view helper. This view helper takes in three variables, a component name, component props and component options and returns a div DOM node with the properties _data-preact-class_ and _data-preact-props_. That’s a lot of confusing words, let me demonstrate.

In your view, you call preact\_component like so:

{% highlight ruby %}
  <%= preact_component(“SimpleButton”, { label: “Start” }) %>
{% endhighlight %}

Where _“SimpleButton”_ is the Preact component name, _label_ is a prop and _“Start”_ is a prop value. And this is what gets rendered when you load the view on a browser:
{% highlight html %}
  <div data-preact-class="SimpleButton" data-preact-props="{'label':'Start'}">
  </div>
{% endhighlight %}

That’s the first part. The second part of the project is an npm package, [preact\_ujs](https://www.npmjs.com/package/preact_ujs). (UJS stands for Unobtrusive JavaScript). This package takes the div rendered above and inflates the named Preact component. This is the complex part but I’ll try make it comprehendible.

Let’s start with the SimpleButtonComponent, defined as such:
{% highlight js %}
  import { h, Component } from "preact"
  
  class SimpleButton extends Component {
    render (props, state) {
      return (<button>{props.label}</button>)
    }
  }
{% endhighlight %}

The component returns a button DOM element with the property _label_ as the child node. Now, back to preact\_ujs. This package has a function [mountComponent](https://github.com/UseHover/preact-rails/blob/master/preact_ujs/index.js#L19) where all the action happens. It starts by finding all DOM components with the attribute _data-preact-class,_ loops through each of them [finding the component class definition](https://github.com/UseHover/preact-rails/blob/master/preact_ujs/index.js#L25) and [parsing the props](https://github.com/UseHover/preact-rails/blob/master/preact_ujs/index.js#L27). Finally, [it renders the component](https://github.com/UseHover/preact-rails/blob/master/preact_ujs/index.js#L28), that’s it!

That’s the inner workings of the gem, you can take a closer look on the [github repo](https://github.com/UseHover/preact-rails). We have complete setup instructions on the README.

We used this gem to test Preact and we loved it. We really didn’t need all the rich features offered by React so the initial proposition attracted us. Three sprints later we launched our new dashboard. So far user feedback has been positive, our developers love it!

We’re still building on our dashboard, powered by our handy little gem. As we build we’ll be making improvements on the gem, starting with tests and TypeScript support. Pull requests are welcome!

Originally posted on the [Hover blog](https://blog.usehover.com/preact-rails-gem--preact-integration-for-ruby-on-rails-/).