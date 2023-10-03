# Elastic Daily Bytes S05E02: Text Analysis

## Setup

Open:

* <https://elastic-daily-bytes-s05.kb.us-central1.gcp.cloud.es.io:9243/app/dev_tools#/console>
* <https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html>

## Introduction

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "_source": {
    "includes": [
      "title",
      "category_name",
      "question.text",
      "question.author",
      "solution.text",
      "solution.author"
    ]
  }
}
```

We have a `title` field. Let's run some basic search on it. We will be covering text search in more details tomorrow.

Search for "grokparsefailure":

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "size": 30, 
  "_source": {
    "includes": [
      "title"
    ]
  },
  "query": {
    "match": {
      "title": "grokparsefailure"
    }
  }
}
```

If we look at the resultset we can see titles containing "grokparsefailure". But do we have also "_grokparsefailure"?

```json
GET bytes-discuss/_search?track_total_hits=true
{
  "size": 30, 
  "_source": {
    "includes": [
      "title"
    ]
  },
  "query": {
    "match": {
      "title": "_grokparsefailure"
    }
  }
}
```

Yes. We have other documents here. I'd expect as a end user to be able to retrieve both resultsets all together. The reason it does not match both terms is because of the text analysis. That's a critical part when doing text based search, so let's dive a bit into it.

## How the title is analyzed?

```json
POST bytes-discuss/_analyze
{
  "field": "title",
  "text": "_grokparsefailure"
}
POST bytes-discuss/_analyze
{
  "field": "title",
  "text": "grokparsefailure"
}
```

This gives:

```json
{
  "tokens": [
    {
      "token": "_grokparsefailure",
      "start_offset": 0,
      "end_offset": 17,
      "type": "<ALPHANUM>",
      "position": 0
    }
  ]
}
{
  "tokens": [
    {
      "token": "grokparsefailure",
      "start_offset": 0,
      "end_offset": 16,
      "type": "<ALPHANUM>",
      "position": 0
    }
  ]
}
```

What we see here is how the text will be actually indexed at index time and searched at search time. That is explaining why `_grokparsefailure` is not matching `grokparsefailure` and vice versa.

## What is an analyzer?

The [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html) tells it in details.

It's a combination of:

* zero or more character filters. Those are responsible to transform the stream of characters by adding, removing or replacing characters, or to strip HTML elements from the text. Which is interesting in our case as you might have noticed that the question and the solution might be in HTML.
* one tokenizer. The Tokenizer splits the stream into tokens depending on its implementation.
* zero or more token filters. They are working on the token stream and they can modify, remove or add new tokens.

## Standard analyzer

Before jumping into that, we can look at the [default list of analyzers](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html) that we have built-in.

The [standard analyzer documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html) describes at the end how it is internally composed with an example in case you want to start from it to build your own custom analyzer.

If we execute the standard analyzer, we are seeing:

```json
POST _analyze
{
  "analyzer": "standard",
  "text": [ "_grokparsefailure problem", "grokparsefailure issue" ]
}
```

```json
{
  "tokens": [
    {
      "token": "_grokparsefailure",
      "start_offset": 0,
      "end_offset": 17,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "problem",
      "start_offset": 18,
      "end_offset": 25,
      "type": "<ALPHANUM>",
      "position": 1
    },
    {
      "token": "grokparsefailure",
      "start_offset": 26,
      "end_offset": 42,
      "type": "<ALPHANUM>",
      "position": 2
    },
    {
      "token": "issue",
      "start_offset": 43,
      "end_offset": 48,
      "type": "<ALPHANUM>",
      "position": 3
    }
  ]
}
```

This is identical with:

```json
POST _analyze
{
  "tokenizer": "standard",
  "filter": [
    "lowercase"       
  ],
  "text": [ "_grokparsefailure PROBLEM", "grokparsefailure issue" ]
}
```

So? What can we do to support both ways of writing "grokparsefailure"?

## Choose the tokenizer

I can replace the standard tokenizer with a `char_group` tokenizer:

```json
POST _analyze
{
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase"       
  ],
  "text": [ "_grokparsefailure PROBLEM", "grokparsefailure issue" ]
}
```

That will give what we are expecting.

## HTML content

To deal with HTML content, we probably need to get rid of the HTML elements.

```json
POST _analyze
{
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase"       
  ],
  "text": "I have a <b>BIG</b> issue!"
}
```

This is producing a strange `<b>big<` token. Obviously, if we index that as is, it won't produce the expected results. Adding `html_strip` character filter will definitely help:

```json
POST _analyze
{
  "char_filter": [
    "html_strip"
  ],
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase"       
  ],
  "text": "I have a <b>BIG</b> issue!"
}
```

To understand all the steps made by the analyzer, you can use `"explain": true` in the analyze API:

```json
POST _analyze
{
  "explain": true,
  "char_filter": [
    "html_strip"
  ],
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase"       
  ],
  "text": "I have a <b>BIG</b> issue!"
}
```

## Searching for issues

```json
POST _analyze
{
  "char_filter": [
    "html_strip"
  ],
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase"
  ],
  "text": ["issue", "issues"]
}
```

We can see that `issues` and `issue` are not matching.

We can use a stemmer for this:

```json
POST _analyze
{
  "char_filter": [
    "html_strip"
  ],
  "tokenizer": {
    "type": "char_group",
    "tokenize_on_chars": [
      "whitespace",
      "punctuation"
    ]
  },
  "filter": [
    "lowercase",
    {
      "type": "stemmer",
      "language": "english" 
    }
  ],
  "text": ["issue", "issues"]
}
```

This now gives `issue` in both cases so that will match.

## Searching within a path hierarchy

```json
POST bytes-discuss/_analyze
{
  "field": "question.author.avatar_template",
  "text": "/user_avatar/discuss.elastic.co/dadoonet/{size}/44959_2.png"
}
POST _analyze
{
  "tokenizer": "keyword",
  "text": "/user_avatar/discuss.elastic.co/dadoonet/{size}/44959_2.png"
}
```

The text is seen as a complete String but if we want to search from the root of the path for something like `/user_avatar/discuss.elastic.co/dadoonet`, this won't match.

Instead we can use a `path_hierarchy` tokenizer:

```json
POST _analyze
{
  "tokenizer": "path_hierarchy",
  "text": "/user_avatar/discuss.elastic.co/dadoonet/{size}/44959_2.png"
}
```

`/user_avatar/discuss.elastic.co/dadoonet` is one of the produced tokens so this will now match.

## Autocompletion

If I want to search while I'm typing the begining of the `username` field:

```json
POST _analyze
{
  "analyzer": "simple",
  "text": ["dado"]
}
```

It will produce `dado` but I do have `dadoonet` in the dataset. So let's use a `edge_ngram` based analyzer:

```json
POST _analyze
{
  "tokenizer": {
    "type": "keyword"
  },
  "filter": [
    "lowercase",
    { 
      "type": "edge_ngram",
      "min_gram": 2,
      "max_gram": 5
    }
  ],
  "text": ["dadoonet"]
}
```

This is producing a list of tokens where I can find `dado` so that will match.

## Closure

I have applied most of those ideas and I reindexed the whole dataset in a new index named `bytes-discuss-02`. I can check that searching for `_grokparsefailure` in the `title` field now gives 40 results instead of 24.

```json
GET bytes-discuss-02/_search
{
  "_source": {
    "includes": [
      "title"
    ]
  },
  "query": {
    "match": {
      "title": "_grokparsefailure"
    }
  }
}
GET bytes-discuss/_search?track_total_hits=true
{
  "_source": {
    "includes": [
      "title"
    ]
  },
  "query": {
    "match": {
      "title": "_grokparsefailure"
    }
  }
}
```

Tomorrow we will be covering the most important queries you need to know about text search. So don't forget to subscribe to this channel!
