---
layout: post
title: The curious case of navigating GitHub repositories
---

Reading a lot of markdown pages on GitHub, I've soon started to miss a way to navigate larger pages more easily. Of course, the obvious solution was to reinvent the wheel and create a [Chrome plugin](https://chrome.google.com/webstore/detail/github-markdown-navigator/mbigiofcajmplcghpdjmapoafafbcmen) to inject a TOC (table of content) into markdown files hosted in GitHub repositories. No worries, this isn't going to be a blog post about writing a chrome plugin. The approach seemed to be straight forward: Traverse the already rendered markdown text and add links to each subtitle on a dedicated location on the page, voilà! But somehow, this simple idea made me doubt my sanity.

After some coding I was happy with the parsing logic and the layout of the plugin. However, for some weird reason, the plugin stopped working once I started navigating around on GitHub and visiting different files in a repository. It just didn't make any sense to me as the plugin code should run on every page load and the code was working just fine when reloading a page using F5. What was this foul sorcery?

```
// this should be called on every single page loaded:
document.onload = function(){
    showToc();
}
```

Of course black magic means there must be JavaScript involved. I will spare you with the details about my debugging process but here's what I found shortly before giving up altogether: When navigating to different files in my GitHub repository (via HTML links), the browser wouldn't load the new HTML document and do its things (including raising DOM events the plugin was listening to) but rather just replace the content via JavaScript "silently" in the background.

After doing some research it turns out that GitHub uses a project called [PJAX](https://github.com/defunkt/jquery-pjax).

> From a user’s perspective, a pjax load is a page load, but from the browser’s perspective it’s just another ajax request
> (https://github.blog/2015-05-19-browser-monitoring-for-github-com/)

So, PJAX stands for pushState + AJAX. PushState is an [API to manipulate a browser's history](https://developer.mozilla.org/en-US/docs/Web/API/History_API#The_pushState()_method). Together, all this JavaScript code will emulate the default navigation behavior of browsers while loading and replacing only partial sections of a page. According to the project site, this reduces unnecessary JS & CSS processing and avoids full layout renderings to achieve faster page loads.

This finally explained why my plugin was broken: Since the navigation between pages on GitHub wasn't fully handled by the browser it wouldn't trigger my plugin code (depending on the `onload` DOM event). Luckily, PJAX [emits custom events](https://github.com/defunkt/jquery-pjax#events) you can subscribe to. Adjusting my plugin to subscribe to these events solved my issue:

```
// PJAX emits a pjax:end event once the content has been replaced.
document.addEventListener("pjax:end", function(){
    showToc();
});
```

I was happy that I got my plugin working. However, I have mixed feelings about PJAX. I disliked the approach of replacing the standard behavior with site-specific JavaScript code. But I realized that this was more or less the same thing SPAs (Single Page Applications) are doing for years and it seems to be mostly fine there. So maybe my aversion just came from the fact that GitHub's PJAX caught me off guard? It would be unfair to blame them for using techniques others praise so much. On the other hand, SPAs use these techniques to build complex, stateful UIs, not primarily as a performance optimization like in GitHub's case, so there is some difference after all. Given that we're talking mostly about CPU bound performance optimizations which save a few processor cycles, I'm not sold on the benefit of this approach given its level of intrusion. If you're thinking about something like PJAX to optimize your site performance, consider checking all IO bound optimizations (loading scripts, images, partial pages, REST APIs, ...) first as these are the biggest performance offenders. It seems the PJAX project is no longer maintained, so I guess we won't see many sites adopting this specific toolkit at least.