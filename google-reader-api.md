# Google Reader API

**Note** : This document is totally unofficial. You should **not** rely
on anything on this document is you **need** an exact information.  
Google Reader API has not officially been released. This document has
been made mainly by reverse-engineering the protocol.

## Requirements

Google Reader API requires:

- http client
- GET and POST support
- Cookie support
- https client

https is only required for identification with Google to get the string
called **SID**. You can rely on an external tool to connect with https
and give you SID. If you do, https won’t be required.

Cookie support is only required to pass to all pages the current
**SID**, proof of identification. If are able to change add lines in
headers, cookie support is not required anymore.

Google Reader API _may_ require:  

- http client  
- GET and POST support  
- external tool to get **SID** (using https)  
- putting **SID** into header

## Glossary

* **SID** : Session ID. SID is generated each time you login to Google
(any service). The SID is valid until you logout.  
* **user ID** : a 20 digits string used by google reader to identify a
user. You don’t really need to know it. You can always do things a way
that user ID is not needed. Usually, when you need that information,
just replace it by **-** and the user ID for current logged user will be
used. The user ID never change for a user.  
* **token** : A token is a special 57 chars string that is used like a
session identification, but that expire rather quickly. You usaully need
a token for direct api calls that change informations. The token is
valid for few minutes/hours. If API call fail (doesn’t return OK), you
just have to get another token.  
* **client ID** : A string that identify the client used. I suppose
it’s for logging/stat purpose, or perhaps to make some adjustment for
some clients, but I doubt so. old GoogleReader interface use `’lens’`,
new one use `’scroll’`. The writer of this document use
`’contact:`_name_`-at-`_host_`’` for interactive test, and use
string like `’pyrfeed/0.5.0’` for the software *pyrfeed* in version
*0.5.0* (like classical identification strings for unix softwares).  
* **item*/*entry** : Sometimes called *item*, sometimes called
*entry*, the item is the base element of a feed. An item usually contain
a text, a title and a link, but can contain other properties. An
RSS/Atom aggregator aggregates items. (Note: *item* is the RSS term,
while *entry* is the Atom term).


## Identification

**Note**: According to [Mihai Parparita in Google Reader Support Group](http://groups.google.com/group/Google-Labs-Reader/browse_frm/thread/73a0aed708bcd005),
“Authentication is one of the reason why the API hasn’t been released
yet”. You should expect big changes before Google Reader release
official API.

To login, you need to post with https on
`https://www.google.com/accounts/ClientLogin` with the following POST
parameters:

| **POST parameter name** | **POST parameter value**                |
|-------------------------|-----------------------------------------|
| service                 | `’reader’` (!)                          |
| Email                   | *your login*                            |
| Passwd                  | *your password*                         |
| source                  | *your client string identification* (!) |
| continue                | `’http://www.google.com/’` (!)          |

Of course, *your login* and *your password* are login and password you
usually use to identify interactively to Google.

(!) : Those parameters are said to be optional (
http://code.google.com/p/pyrfeed/issues/detail?id=10 ). I didn’t really
tested. It’s just how “browsers” identifies themselves. Note that I
consider *source* and *service* as a being informative parameters for
google, so I feel like I **need** to provide them, event if they are not
required. Note that I have no idea about what Google or the Google
Reader team really think about that. Perhaps they prefer “nothing but a
crafted information”, or perhaps they don’t care. Who knows…

There is no official rule *your client string identification*, see
**client ID** in glossary for more informations.

The POST action will return you a text file, containing lines of the
form :

`*key*=*value*`

You need to extract the *value* for the *key* named `SID`

You then have to add yourself a cookie (well, it look like Google
doesn’t add it itself) with the following properties :

|---------|---------------|
| name    | `SID`         |
| domain  | `.google.com` |
| path    | `/`           |
| expires | `1600000000`  |

If you don’t have a http client API that support cookies, you can just
add header lines that simulate this cookie in all other requests. This
should be the only thing for which cookies are really needed.


## The three layers for feed aggregators

When you’re writing a feed aggregator, you need to write three
different layers:

1.  **Layer 1** : The layer that parse feeds. It’s not the easiest job.
“But, it’s just xml, it should be easy”. It’s not. It’s just xml. It’s
just 10 different and incompatible xml formats (9 RSS formats according
to Mark Pilgrim and 1 Atom format). You also perhaps need to understand
all non standard feeds that mix some features from different
standards.  
2. **Layer 2** : The database layer. Once you’ve parsed your feed, you
need to store it in a database, and and interesting things like “items
read”, etc.  
3. **Layer 3** : The user interface.

Google Reader offer in fact access to **layer 1** only, or **layer 1+2**
or **layer 1+2+3**.

You can have access to **layer 1** only. Feeds are parsed by Google
Reader, and Google Reader give you access to a new Atom feed that
contains same data as the original feed, but always with the same output
format : Atom.

You can have access to **layer 1+2**. This documents purpose is about how
to access to **layer 1+2** from Google Reader in order to create your own
**layer 3**.

Of course, you have access to **layer 1+2+3** because it’s Google
Reader’s main product.

## Url scheme

Except for identification process, all Google Reader url resources
start with `http://www.google.com/reader/`. We’ll explain here direct
subspaces of those urls.

| **URL prefix**                           | **Will be referred as, in this document** | **Description**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|------------------------------------------|-------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `http://www.google.com/reader/atom/`     | */atom/*                                  | All urls starting with this prefix return atom feeds. It’s the (only?) way to acces to feed contents. This is the way to access **layer 1** and **layer 1+2**.                                                                                                                                                                                                                                                                                                                       |
| `http://www.google.com/reader/api/0/`    | */api/0/*                                 | This is the main API entry point. It’s used for items/feeds modifications, like adding a *Star*, deleting a tag, etc. For those modification services, it return either `“OK”` or `“”`. It’s also used to consult some setting lists like list of feeds, list of tags, list of unread counts by feeds/tags, etc. For those read services, it returns an *“object”* that can be either *[http://json.org/ json]* or *xml that look like json*. This is a **layer 2** only zone. |
| `http://www.google.com/reader/view/`     | */view/*                                  | All AJAX interface is done by /view/ urls. AJAX code use */atom/* and */api/0/* as sublayers to do the job. This is the way to access **layer 3**.                                                                                                                                                                                                                                                                                                                                   |
| `http://www.google.com/reader/shared/`   | */shared/*                                | All shared pages use this prefix. You obviously don’t need authentification to use those pages.                                                                                                                                                                                                                                                                                                                                                                                      |
| `http://www.google.com/reader/settings/` | */settings/*                              | The AJAX application to configure all settings. Mainly manipulate informations from */api/0/*. This part of **layer 3**.                                                                                                                                                                                                                                                                                                                                                             |

Atom set of items

*this section is about urls starting with
`http://www.google.com/reader/atom/`

The Google Reader database contains a huge number of items. Some of them
are in your *reading list* (understand, they are accessible in Google
Reader for your account in “*All items*” section, and in your
feeds/tags).

The only way to get information related to an item is to “query an atom
set of items” that contain this item.

All items are or were included in feeds. One way to query items is to
query the original feed.

Item are also associated to categories. tags/labels are categories, but
also the “read” state is also a category.

Another way to query items is to query all items that are associated to
a category.

|         |              |                     | **Set of items suffix** | **Description**                                                                                                                                                                      |
|---------|--------------|---------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `feed/` |              |                     | *url of a feed*         | The url to query a specific feed. It’s Google Reader way to access to **layer 1** only. **Note** : This service is not related to an account and can be access without registration. |
| `user/` | _user ID_`/` | `label/`            | *label name*            | This is the suffix to access to all items with a specific label                                                                                                                      |
| `user/` | _user ID_`/` | `state/com.google/` | *state*                 | This is the suffix to access to all items with a specific state like **`read`**, **`starred`**, etc.                                                                                 |

You can use `-` as your **user ID**, it will use the **user ID** for
your currently identified account.

| **State name**              | **State meaning**                                                                                                                               |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
| `read`                      | A read item will have the state *read*                                                                                                          |
| `kept-unread`               | Once you’ve clicked on “keep unread”, an item will have the state *kept-unread*                                                                 |
| `fresh`                     | When a new item of one of your feeds arrive, it’s labeled as *fresh*. When (*need to find what remove fresh label*), the fresh label disappear. |
| `starred`                   | When your mark an item with a star, you set it’s *starred* state                                                                                |
| `broadcast`                 | When your mark an item as being public, you set it’s *broadcast* state                                                                          |
| `reading-list`              | All you items are flagged with the *reading-list* state. To see all your items, just ask for items in the state `reading-list`                  |
| `tracking-body-link-used`   | Set if you ever clicked on a link in the description of the item.                                                                               |
| `tracking-emailed`          | Set if you ever emailed the item to someone.                                                                                                    |
| `tracking-item-link-used`   | Set if you ever clicked on a link in the description of the item.                                                                               |
| `tracking-kept-unread`      | Set if you ever mark your read item as unread.                                                                                                  |

If you need to query a set of items in an atom format, just query
`http://www.google.com/reader/atom/` followed by the set of items
suffix.

For example, if you want to access to Google Reader’s rewriting of the
feed http://xkcd.com/rss.xml , you can query
`http://www.google.com/reader/atom/feed/http://xkcd.com/rss.xml`. This
can be done whether you are identified or not.  
If you want to query all your last read items, you can query
`http://www.google.com/reader/atom/user/-/state/com.google/read`.

Each atom set contains by default 20 items. You can change that, and
other behaviour by adding parameters to the query.

| **GET parameter name** | **python Google Reader API name** | **parameter value**                                                                                                                                                                                                                                                                                |
|------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `n`                    | *count*                           | Number of items returns in a set of items (default 20)                                                                                                                                                                                                                                             |
| `client`               | *client*                          | The default client name (see *client* in glossary)                                                                                                                                                                                                                                                 |
| `r`                    | *order*                           | By default, items starts now, and go back time. You can change that by specifying this key to the value `o` (default value is `d`)                                                                                                                                                                 |
| `ot`                   | *start_time*                      | The time (unix time, number of seconds from January 1st, 1970 00:00 UTC) from which to start to get items. Only works for order r=o mode. If the time is older than one month ago, one month ago will be used instead.                                                                             |
| `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered.                                                                                                                                                                                                        |
| `xt`                   | *exclude_target*                  | another set of items suffix, to be excluded from the query. For example, you can query all items from a feed that are not flagged as read. This value start with `feed/` or `user/`, not with `!http://` or `www`                                                                                  |
| `c`                    | *continuation*                    | a string used for *continuation* process. Each feed return not all items, but only a certain number of items. You’ll find in the atom feed (under the name `gr:continuation`) a string called continuation. Just add that string as argument for this parameter, and you’ll retrieve next items.   |

**Note** : *continuation* has no meaning, it’s just a string to help you
find next items. You should not rely on its value to do anything else
than that (even if this document will explain how that continuation is
generated).

**Example** :  
All the 17 first items items from xkcd.com main feed that are not read
can be found on the url:
`http://www.google.com/reader/atom/feed/http://xkcd.com/rss.xml?n=17&ck=1169900000&xt=user/-/state/com.google/read`

# API

*this section is about urls starting with
`http://www.google.com/reader/api/0/`*

There are two kinds of API commands:  
* edit commands  
* list commands

The number 0 is probably the API version number. Using that number, it
will allow Google Reader to change API while stile maintaining an old
API for quite some time.

## Edit API

To edit anything in the Google Reader database, you need a token (see
glossary).  
To get a token, just go to http://www.google.com/reader/api/0/token .
This url will return a string containing 57 chars. It’s the token.

The token url takes optional GET arguments:

| **GET parameter name** | **python Google Reader API name** | **parameter value**                                                                         |
|------------------------|-----------------------------------|---------------------------------------------------------------------------------------------|
| `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered. |
| `client`               | *client*                          | The default client name (see *client* in glossary)                                          |

All edit commands use `POST` to retrieve information (note that GET
won’t work) but they also take a `GET` argument.

| **GET parameter name** | **python Google Reader API name** | **parameter value**                                |
|------------------------|-----------------------------------|----------------------------------------------------|
| `client`               | *client*                          | The default client name (see *client* in glossary) |

All edit commands will return either an empty string if failled, either
the string “`OK`”. If failed, your token has perhaps expires. You can
just try to get a new token. If it still doesn’t return OK, it’s a
failure.

Table of POST arguments for the `subscription/edit` edit call.

| **API call function**   | **POST parameter name** | **python Google Reader API name** | **parameter value**                                                                                                                       |
|-------------------------|-------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| **`subscription/edit`** |                         |                                   |                                                                                                                                           |
|                         | `s`                     | *feed*                            | The subscription feed name, in the form `feed/`*…*                                                                                        |
|                         | `t`                     | *title*                           | The subscription title, used when adding a new subscription or when changing a subscription name                                          |
|                         | `a`                     | *add*                             | A label to add (a label on a subscription is called a folder) in the form `user/`*…*                                                      |
|                         | `r`                     | *remove*                          | A label to remove (a label on a subscription is called a folder) in the form `user/`*…*                                                   |
|                         | `ac`                    | *action*                          | The actions to do. Know values are `edit` (to add/remove label/forlder to a feed), ‘subscribe’, ‘unsubscribe’                             |
|                         | `token`                 | *token*                           | The mandatory up to date token                                                                                                            |
| **`tag/edit`**          |                         |                                   |                                                                                                                                           |
|                         | `s`                     | *feed*                            | The tag/folder name seen as a feed                                                                                                        |
|                         | `pub`                   | *public*                          | A boolean string `true` or `false`. When `true`, the tag/folder will become public. When `false`, the tag/folder will stop being public.  |
|                         | `token`                 | *token*                           | The mandatory up to date token                                                                                                            |
| **`edit-tag`**          |                         |                                   |                                                                                                                                           |
|                         | `i`                     | *entry*                           | The item/entry to edit, in the form `tag:google.com,2005:reader/item/`*…* ( it’s the xml id of the `entry` tag of the atom feed)          |
|                         | `a`                     | *add*                             | A label/state to add (a label on an item/entry is called a tag) in the form `user/`*…*                                                    |
|                         | `r`                     | *remove*                          | A label/state to remove (a label on an item/entry is called a tag) in the form `user/`*…*                                                 |
|                         | `ac`                    | *action*                          | The actions to do. Know value is `edit` (to add/remove label/forlder to a feed)                                                           |
|                         | `token`                 | *token*                           | The mandatory up to date token                                                                                                            |
| **`disable-tag`**       |                         |                                   |                                                                                                                                           |
|                         | `s`                     | *feed*                            | The tag/folder name seen as a feed                                                                                                        |
|                         | `ac`                    | *action*                          | The actions to do. Know value is `disable-tags` (to remove a tag/folder)                                                                  |
|                         | `token`                 | *token*                           | The mandatory up to date token                                                                                                            |

Examples :  
To subscribe a new feed (for example http://xkcd.com/rss.xml), you can
call:

`http://www.google.com/reader/api/0/subscription/edit?client=contact:myname-at-gmail`

with POST arguments :
`http://xkcd.com/rss.xml&ac=subscribe&token=_here-put-a-valid-token_`

To add that feed in a folder (for example “comics”), you can call:

http://www.google.com/reader/api/0/subscription/edit?client=contact:myname-at-gmail

With POST arguments:
`http://xkcd.com/rss.xml&ac=edit&a=user/-/label/comics&token=_here-put-a-valid-token_`

Open questions that needs to be fixed :  
* Are `tag/edit` and `edit-tag` just aliases ?  
* Why does removing a tag/folder doesn’t take an action while every
other calls take actions ?  
* Why is there several urls, it seems redundant with the action
parameter ?

## List API

All those calls can be used with GET requests.

|  **API call function**  | **GET parameter name** | **python Google Reader API name** | **parameter value*/*API call description**                                                                    |
|-------------------------|------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------------|
| **`tag/list`**          |                        |                                   | Get the tag list and shared status for each tag.                                                              |
|                         | `output`               | *output*                          | The format of the returned output. may be ‘json’ or ‘xml’                                                     |
|                         | `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered.                   |
|                         | `client`               | *client*                          | The default client name (see *client* in glossary)                                                            |
| **`subscription/list`** |                        |                                   | Get the subscription list and shared status for each tag.                                                     |
|                         | `output`               | *output*                          | The format of the returned output. may be ‘json’ or ‘xml’                                                     |
|                         | `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered.                   |
|                         | `client`               | *client*                          | The default client name (see *client* in glossary)                                                            |
|                         |                        | **`preference/list`**             | Get the preference list (configuration of the account for GoogleReader).                                      |
|                         | `output`               | *output*                          | The format of the returned output. may be ‘json’ or ‘xml’                                                     |
|                         | `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered.                   |
|                         | `client`               | *client*                          | The default client name (see *client* in glossary)                                                            |
| **`unread-count`**      |                        |                                   | Get all the information about where are located (in term of subscriptions and tags/folders) the unread items. |
|                         | `all`                  | *all*                             | ‘true’ if whole subscriptions/tags are required. (**TODO**: *Needs to check other values*)                    |
|                         | `output`               | *output*                          | The format of the returned output. may be ‘json’ or ‘xml’                                                     |
|                         | `ck`                   | *timestamp*                       | current time stamp, probably used as a quick hack to be sure that cache won’t be triggered.                   |
|                         | `client`               | *client*                          | The default client name (see *client* in glossary)                                                            |

Viewer

**this section is about urls starting with
`http://www.google.com/reader/view/`**

All url starting with `http://www.google.com/reader/view/` are html pages
that use AJAX code to show atom feeds found from
`http://www.google.com/reader/atom/`.

You can append to the base url any *set of items suffix* to view only
that set of items. Note however that GET parameters are not valid (in
fact are ignored) for those urls.

You can browse directly all your items labeled “important” by going to
`http://www.google.com/reader/view/user/-/label/important`

You can browse directly all items from xkcd main feed by going to
http://www.google.com/reader/view/feed/http://xkcd.com/rss.xml even if
you didn’t subscribed to it (in which case there will be a button
“Subscribe” on the top of the screen). Note however that if you’re not
identified, you’ll browse the feed using the old interface (lens) and
not the new on (scroll).

Misc

- [ ] TODO: Text needs to be written*  
- [ ] TODO: mainly /share/*

## References

- [First document being considered as an unofficial Google Reader API.](http://www.niallkennedy.com/blog/archives/2005/12/google_reader_a.html)
- [Mark Pilgrim explaining the mess with all RSS formats](http://www.xml.com/pub/a/2002/12/18/dive-into-xml.html)

!["broken image"](http://pyrfeed.yi.org/image.png)
