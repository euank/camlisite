# Serving websites from Camlistore

This document describes a design for serving static website content from a camlistore server or set of servers.

## Goals

The following are goals of this design:

### Compatible with existing directory/file hierarchy

Static files, including html documents, stylesheets, images, and more, should all be able to be served easily.

Furthermore, it should be compatible with the existing directory and file
concepts that a semi-formalized scheme exists for.

In other words, the result of `camtool sync` should be easy to serve as a static site.

### Supports redirects rules

Redirects are an important part of fighting link rot despite shifting site structure.

It should be easy to create once-off redirects (of the form /location/a -> location/b).

It should also be possible to create more complex rewrite rules (of the form /location/.\* -> /rewritten/to/base/).

### Supports host independent of content

The host (e.g. example.com) should be independent of the static content. Notably, it should be easy to have content at one location (e.g. staging.example.com) and then trivially back it by another host.

In addition, it should be easy to have a many-to-one mapping of host to content (e.g. www1.example.com and www2.example.com being backed by the same camlistore content).

## Non-goals

### Templating

At least not at first. Not for a whle.

### Being a reverse-proxy

## Design

The primary pieces introduced are the following:

1. A 'camlisite' http server component
1. A small number of additional camliType schemas.

### Camlisite

The camlisite binary provides an http interface. It acts as a client of a camlistore server and is configured with a normal client-config.json.

The camlisite binary, by default, will serve all sites which it discovers on
the target server. However, it may be configured to only serve a specific set
of names, or with a given filter of tags to serve.

These tags would specifically be permanodes containing those tags which point to websites.

### New 'camliType's

The most important new schema introduced is the 'website' type. It has the following structure:

```json
{"camliVersion": 1,
 "camliType": "website",
 // The hostname of this website
 "host": "example.com",
 // A list of references.
 // Each reference may be a static-set, a directory, a file, a website, a
 // redirect, a rewrite, or any permanode with a camliPath attribute.
 // order is meaningful as each request will be addressed by references in
 // order.
 // Subsequently, the last target should typically be a rewrite to 
 // a 404 path, or a file that acts as the 404 page.
 "targets": ["sha1-543fbdfdbcb1297af8a4dc7d299c0cb90e2bea0f", ..]
}
```

In addition, the following types are introduced:

```json
{"camliVersion": 1,
 "camliType": "redirect",
 "from": "sha1-abc", // optional; if omitted this redirect will be unconditional
 "to": "sha1-def"
}
```

```json
{"camliVersion": 1,
 "camliType": "uriRef",
 "uri": "https://google.com"
}
```

```json
{"camliVersion": 1,
 "camliType": "rewrite",
 "from": "sha1-abc", // optional; if omitted this rewrite will be unconditional
 "to": "sha1-def"
}
```


## Example

The following shows how an example site might be represented. This site
specifically is contrived to demonstrate as many features as possible.

The site has the following routes:

```
Key: => is a rewrite
     -> is a redirect

example.com
  / => /index.html
  /about.html
  /404.html
  
  /about/ => /about.html
  
  /blog/ -> blog.example.com

  /.* => /404.html

blog.example.com
  / => index.html
  ...
```

This can be represented in the above with:
