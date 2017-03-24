---
layout: post
title: Positioning a fixed header below the Sitecore Experience Editor ribbon
date: 2017-03-24 19:35:12.000000000 +11:00
type: post
published: true
status: publish
tags:
- sitecore
author:
  login: richardszalay
  email: richard@richardszalay.com
  display_name: Richard Szalay
  first_name: ''
  last_name: ''
---

This post attempts to explain how to integrate into the Sitecore Experience Editor (Page Editor) to move your website's `position: fixed` header so that it is always underneath the Experience Editor's ribbon. The theory could also be applied to `position: sticky` elements.

Let's say we have this (beautifully designed) header:

<img src="{{ site.baseurl }}/assets/sc-exp-h-sample-header.png" alt="Sample header" />

Now if that header is styled with "position: fixed", it's going to appear behind the ribbon in the Experience Editor:

<img src="{{ site.baseurl }}/assets/sc-exp-h-normal-wrong.png" alt="Position position header appearing underneath the Experience Editor ribbon" />

This makes the navigation nearly impossible to use without closing the ribbon. When the ribbon is expanded, it's even worse:

<img src="{{ site.baseurl }}/assets/sc-exp-h-expanded-wrong.png" alt="Position position header appearing underneath the expanded Experience Editor ribbon" />

## Understanding the ribbon's layout

In order to fix this, we need to understand how Sitecore _tries_ to push down page content when the ribbon is open.

The Experience Editor ribbon UI is actually presented from within an iframe, presumably to avoid conflicting styles or scripts. The (unfixed) page content ends up being pushed down because of an empty div element, called `scCrossPiece`, that is sized in sync with the iframe content. Here's a crude visualisation:

<img src="{{ site.baseurl }}/assets/sc-exp-h-sccrosspiece.png" alt="Visualisation of the ribbon UI iFrame and the scCrossPiece element that represents the former's size" />

However, because `position: fixed` elements are positioned based on absolute coordinates, they aren't affected by `scCrossPiece`. The solution, therefore, is to detect changes to the scCrossPiece container and update the position of the header.

## Client pipelines to the rescue

We could simply include Experience Editor-aware javascript in our application code, but that feels like a code smell. Specifically, it's a violation of separation of concerns and it exposes the fact that our system is based on Sitecore. 

Luckily, there is a "Sitecore approved" way of solving this type of problem and that is via "client pipelines". Client pipelines work similarly to Sitecore's server side pipelines but are written in JavaScript. In our case, we want to integrate with the "InitializePageEdit" pipeline as that runs when the Experience Editor UI starts up.

First, let's create our script that will be run by the pipeline. Since there are no events fired when the ribbon is resized, we'll use [MutationObserver](https://developer.mozilla.org/en/docs/Web/API/MutationObserver) to be informed when the div's style attribute changes. MutationObserver is [supported in the latest browsers and IE11](http://caniuse.com/#feat=mutationobserver), but there's at least one [polyfill](https://github.com/Polymer/MutationObservers) that brings support to IE9 and above.

{% gist f8fe2c90f21132ad7fb7d26d71c1faf3 PositionFixedNavBar.js %}

Next we need to create a new `PipelineProcessor` item named `PositionFixedNavBar` in Sitecore's core database under `/sitecore/client/Applications/ExperienceEditor/Pipelines/InitializePageEdit`. The `ProcessorFile` field should be set to an absolute path to the javascript file (eg. "/assets/js/PositionFixedNavBar.js").

Now, when we load the experience editor the nav is always positioned below the ribbon and the code involved isn't present in CD environments!

<img src="{{ site.baseurl }}/assets/sc-exp-h-normal-right.png" alt="Sample header" />

<img src="{{ site.baseurl }}/assets/sc-exp-h-expanded-right.png" alt="Sample header" />

## Bonus round: "Sitecore Extensions" support

[Sitecore Extensions](https://chrome.google.com/webstore/detail/sitecore-extensions/aoclhcccfdkjddgpaaajldgljhllhgmd) is a Chrome extension that injects additional UI elements into the Sitecore client to make certain tasks easier. One of these features is a "hide ribbon" button, which is usually more convenient than "closing" the ribbon as the latter requires a page refresh. Unfortunately, the extension hides the ribbon via a CSS class and our implementation so far isn't aware of that, so it ends up looking like this:

<img src="{{ site.baseurl }}/assets/sc-exp-h-collapsed-wrong.png" alt="Sample header" />

Let's can tweak our implementation to take the CSS class into consideration:

{% gist f8fe2c90f21132ad7fb7d26d71c1faf3 PositionFixedNavBar-SC-Ext.js %}

Now our header is positioned correctly, even when the ribbon is hidden:

<img src="{{ site.baseurl }}/assets/sc-exp-h-collapsed-right.png" alt="Sample header" />