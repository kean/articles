---
layout: post
title: "Client-Side Search"
subtitle: How to build client-side search with lunr.js
description: How to build client-side search with lunr.js
date: 2021-04-09 09:00:00 -0500
category: programming
tags: programming
permalink: /post/client-side-search
uuid: 73543c14-bc64-4bf3-8913-210e2dbf1268
image:
  path: /images/posts/client-side-search/cover.png
  height: 1280
  width: 640
---

<div class="BlogVideo NewScreenshot">
<video controls muted playsinline preload="auto">
	<source src="{{ site.url }}/videos/nuke-docs/search.mp4" type="video/mp4">
</video>
</div>

The search is for [Nuke Docs]({{ site.url}}/nuke/guides/welcome) is built using [lunr.js](https://lunrjs.com) which is a simple client-side search engine which is relatively easy to use.

## Populating Index

First, you need to populate a search index. In Nuke Docs, an index is a JSON file generated during the Jekyll build process. The file has [front matter](https://jekyllrb.com/docs/front-matter/) to enable special processing where you can use [Liquid](https://jekyllrb.com/docs/front-matter/) tags.

```js
---

---

[
{% raw %}{% assign guides = site.pages | where: 'layout', 'nuke-guide' %}
{% for guide in guides %}
  {
    "title": {{guide.title | jsonify}},
    "content": {{ guide.content | markdownify | jsonify }},
    "link": {{ guide.url | jsonify  }}
  }{% unless forloop.last %},{% endunless %}
{% endfor %}{% endraw %}
]
```

As an output, I get a simple [`search-index.json`](https://kean.blog/nuke/assets/search-index.json) file with an array of pages. When the user presses the search button (or an `f` key), the app fetches it and populates the lunr.js index.

```javascript
function populateSearchIndex(json) {
    let entries = json.flatMap(splitPageIntoChunks);

    searchIndex = lunr(function () {
        this.field('title', { boost: 5 });
        this.field('content');
        this.ref('id');

        for (var i = 0; i < entries.length; i++) {
            const entry = entries[i];
            this.add({ title: entry.title, content: entry.content, id: i });
        }
    });

    for (const entry of entries) {
        const excerpt = truncate(entry.content, 120, true);
        searchEntries.push({ 'title': entry.title, 'link': entry.link, 'excerpt': excerpt });
    }
}
```

## Searching

With the index populated, you can start searching. With lunr.js 2.0, the search simply looks for complete matches of the input words by default. For example, if you want to search for `priorirty`, but only enter `priori`, the results will be empty. So I ended up with the following approach for creating a query:

```javascript
const updatedQuery = query
    .split(" ")
    .map(word => word + '^2 ' + word + '*')
    .join(' ');

const results = searchIndex.search(updatedQuery);
```

I search for a complete match boosting its score (`^2`) and for a prefix using a wildcard (`*`). I also tried adding a bit of fuzziness (`~1`) but was producing inaccurate results with high scores.

## Granular Index

The search works, but only for complete pages: you can't jump to subheadings. Fortunately, I already had [jekyll-anchor-headings](https://github.com/allejo/jekyll-anchor-headings) set up for the table of contents, so all I needed to do was preprocess the pages. I take a page, split it into chunks using simple regex[^5], find anchor links, strip HTML and Liquid tags from content, and voila – now the search index has parts of pages split based on subheadings instead of complete pages.

I don't want to paste the complete code here, but you can always find it on the [website](https://kean.blog/nuke/guides/welcome). As for the UI, you can find it there as well; it was challenging to get the search view to work precisely the way I wanted it to with keyboard navigation, automatic scrolling, nice highlighting, etc.

[^5]: `.content.split(/(<h[^>]*>.*?<\/h.>)/g)`. Parsing HTML with regex is generally not a good idea, but it works well in this controlled environment.

## Conclusion

[lunr.js](https://lunrjs.com) is a great tool with more features that I haven't covered here. Fortunately, it has a website with [documentation](https://lunrjs.com/guides/searching.html) of its own.
