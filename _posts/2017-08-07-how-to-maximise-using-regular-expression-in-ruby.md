---
layout: post
title: How to maximise using Regular Expression in Ruby
date: 2017-08-07
comments: true
categories: [ruby, regular expression, regex]
tags: [ruby, regular expression, regex]
excerpt_separator: <!-- more -->
---

Regular Expression can be overlooked by Ruby developer sometimes. However, if we know how to make use Regular Expression, it could be extremely useful and save a lot of our time dealing with ruby strings problem. In this article, I am going through some basic knowledge of Regular Expression in Ruby and how we can fully utilise it for our benefit.

<!-- more -->

Regular Expression is hard to master and sometimes it can cause us more trouble if we use it wrongly. The good news is that we can learn it slowly and apply it gradually in our project without causing us too much trouble. Let's go through some basic regular expressions that we can quickly grab:

### Ruby match function

We can use Ruby ``match`` function to test if a regular expression match with a string. Simply call the ``match`` function from the regular expression and suppy a string as its parameter. If it does match, an instance of ``MatchData`` would return. Otherwise, a ``nil`` object would be returned instead. For example:

```ruby
/a/.match('a')
# => #<MatchData "a">

/a/.match('b')
# => nil
```

Now we know how to use this match function, let's dive into the list of common regular expressions we could use immediately:

### 1 - Match a single character

```ruby
/a/.match('a')
# => #<MatchData "a">
```

This is as simple as you can see. A single character regular expression will match a single character string.

### 2 - Match an alphabet

```ruby
/[a-z]/.match('d')
# => #<MatchData "d">

/[a-z]/.match('9')
# => nil
```

We are using square brackets to enclose a list of characters ranging from ``a`` to ``z``, hence all single character string with a character from ``a`` to ``z`` would match with this regular expression. Otherwise, it will not match, hence returns a ``nil``.

### 3 - Match a single digit

```ruby
/[0-9]/.match('8')
# => #<MatchData "8">
```

We are again using square brackets to enclose a list of character. However, this time is from character ``'0'`` to character ``'9'``. Hence, this regular expression will match with a single digit string like ``'8'`` in this case.

A shorter form of this regular expression ``[0-9]`` is ``\d``, where ``d`` stands for digit. We can write:


```ruby
/\d/.match('8')
# => #<MatchData "8">
```

### 4 - Match all letters, numbers and underscore (_)

```ruby
/\w/.match('8')
# => #<MatchData "8">
```

``\w`` is another short form of matching all letters, numbers and underscore (_) character. Usually, to achieve this, we would need to write like following:

```ruby
/[a-zA-Z0-9_]/.match('8')
# => #<MatchData "8">
```

The short form ``\w`` does help to make thing look cleaner.

These 4 are pretty basic for starter. Let's move on to something intermediate.

### 5 - Match starting of a line

```ruby
/^a/.match('a')
# => #<MatchData "a">
```

We use ``^`` to denote the starting of a line. The end result is exactly the same as point 1 above but the effect is pretty different. This says that, your testing string must start with ``a`` character to match with the regular expression. This would be useful when you need to check a string start with something. For example, if you want to check a phone number starts with a plus (+) sign.


### 6 - Match ending of a line

```ruby
/a$/.match('a')
# => #<MatchData "a">
```

We use ``$`` to denote the ending of a line. This would be useful if you want to check a string that ends with something. For instance, you can check if a string is an email by checking it ends with ``.com``.

Combining ``^`` and ``$`` will get you a regular expression that sandwich a string:

```ruby
/^a$/.match('a')
# => #<MatchData "a">

/^a$/.match('ab')
# => nil
```

As you can see, the above regular expression would only be able to match single character string ``a`` but not ``ab`` as ``ab`` does not ends by ``a`` character.

### 7 - Match 1 or more character

```ruby
/[a-z0-9]+/.match('rails51')
# => #<MatchData "rails51">
```

We use the operator ``+`` to match 1 or more than 1 character. In the above case, it will match all characters of the string ``rails51``. The regular expression will always try to match as many as possible characters of the string.

### 8 - Match an optional character

```ruby
/[a-z0-9]?/.match('rails51')
# => #<MatchData "r">
```

The operator ``?`` will match an optional character, or in other word, it will try to match at most 1 character. For the above case, it will match the character ``r`` and then stop.

### 9 - Match at least and at most a number of characters

```ruby
/[a-z0-9]{0,5}/.match('rails51')
# => #<MatchData "rails">
```

We use ``{n, m}`` to indicate at least ``n`` characters must be matched and at most ``m`` characters must be matched. So in the case above, we are saying that it has to match up to ``5`` characters, hence resulting in ``rails`` as the matching string.


### 10 - Match exact a number of characters

```ruby
/[a-z0-9]{5}/.match('rails51')
# => #<MatchData "rails">
```

If we want to match exact a number of character, we can use ``{n}`` to indicate the exact characters that must be matched. In the case above, we indicate the string must match exact ``5`` characters hence resuling in ``rails`` as the matching string.

So these are the most common 10 regular expression patterns that I used. Combining these will help to come out with a very sophisticated regular expression. For example:


```ruby
/^[a-z0-9]+@[a-z0-9]+(\.com|info|org)$/.match('john@example.com')
# => #<MatchData "john@example.com" 1:".com">
$1
# => ".com"
```

It looks complicated at first but let's dive in and split it into few components to see that it is pretty simple:


The first part is:

```ruby
^[a-z0-9]+
```

This is pretty straight forward as this indicates that this string must start with an alphabet or digit strings. The ``+`` sign is indicating that this string can have as many characters as possible. This part is used to match ``john`` string.

The second part is:

```ruby
@
```

``@`` this is just a single character saying that the string must has this ``@`` character after having the above first part. This part is used to match ``@`` string

The third part is:

```ruby
[a-z0-9]+
```

This is yet another string with alphabets or digits and can have as many characters as possible. This part is used to match ``example``.

The last part is:

```ruby
(\.com|info|org)$
```

This indicates that the string must end with ``.com`` or ``.info`` or ``.org``. By default ``.`` in regular epxression means match everything. But in this case, we want it to match with the dot (.) character, hence backslash (``\``) must be used.

The bar (``|``) character is used as an or.
This last part is used to match ``.com``

As you can see, a string can be decomposed into multiparts and we can use different regular expression to target each of the part of the compounded string.

Also, in the above regular expression, I am using the parentheses ``(...)``. This is to group the regular expression so that we can extract the string out easily. For instance, the result of the match data above is:

```ruby
# => #<MatchData "john@example.com" 1:".com">
```

You can see the part ``1: "com"`` there to indicate that we can use the variable ``$1``. ``$1`` will bring the value ``.com``. We can do the same thing with the remaining parts of the regular expression to extract out the things that we need. Pretty neat!


```ruby
/^([a-z0-9]+)@([a-z0-9]+)(\.com|info|org)$/.match('john@example.com')
# => #<MatchData "john@example.com" 1:"james" 2:"example" 3:".com">
$1
# => james
$2
# => example
$3
# => .com
```

However, in some case, we do not want to capture the group. We can then use the question mark and colon (``?:``) to exclude the group out from the capturer. For instance:


```ruby
/^[a-z0-9]+@[a-z0-9]+(?:\.com|info|org)$/.match('john@example.com')
# => #<MatchData "john@example.com">
```

One thing I really loved about Ruby 2.4 is that it now provides a function to ``MatchData`` instance so that we can create a hash out of the matched result without accessing the global variable ``$1``, ``$2``, ... Ehem, you all know global variable is bad.


```ruby
# ruby 2.4 only
/^(?<name>[a-z0-9]+)@(?<domain>[a-z0-9]+)(?<ext>\.com|info|org)$/.match('john@example.com')
# => {"name" => "john", "domain" => "example", "ext" => ".com"}
```

This is pretty awesome as so much information can be extracted out just by using 1 simple function ``match``.

### Application

One useful application of regular expression is Ruby on Rails is that you can apply regular expression to write your validation of input string from user. For example, I can write the following regular expression code in Rails to validate Singapore phone number:

```ruby
validates_format_of :phone, with: /^\+65 [0-9]{8}$/
```

Another application is that we can use regular expression to transform a string to another string. For instance, let's say we have an input box that a user can copy a youtube video URL in and our task is to convert it to an embedded iframe later on. The input would be some thing like this:

```ruby
https://www.youtube.com/watch?v=KO8PV6tMXeo
```

Notice the part after ``?v=``. It is actually the ID of the video itself.

The result iframe HTML should be:

```html
<div style="position:relative;height: 0;padding-bottom:56.25%"><iframe src="https:// www.youtube.com/embed/KO8PV6tMXeo?ecver=2" width="640" height="360" frameborder="0" style="position:absolute;width:100%;height: 100%;left:0" allowfullscreen></iframe></div>
```

We actually would only need to somehow extract out the video ID and paste it after the st ``https:// www.youtube.com/embed/xxxxxxxxxxx``. Sound simple as it is but without regular expression, it can be quite hassle whereby we have to split the string out from the initial Youtube video URL and replace it inside the second string. A naive solution would be something like below:

```ruby
str = 'https://www.youtube.com/watch?v=KO8PV6tMXeo'
youtube_video_id = str.split('?v=')[1]
embeded_iframe = %(<div style="position:relative;height: 0;padding-bottom:56.25%"><iframe src="https:// www.youtube.com/embed/{{youtube_video_id}}?ecver=2" width="640" height="360" frameborder="0" style="position:absolute;width:100%;height: 100%;left:0" allowfullscreen></iframe></div>).gsub('{{youtube_video_id}}', youtube_video_id)
```

This works but it is not elegant and does not guarantee 100% success. Imagine, what would happen if inside the embedded iframe string, there is another string ``{{youtube_video_id}}`` somewhere. It could mess up with our ``gsub`` function.

With regular expression, it is quite a breeze for the solution:


```ruby
str = 'https://www.youtube.com/watch?v=KO8PV6tMXeo'
matches = str.match(/v\=(?<video_id>\w+)/)&.named_captures
if matches
	embeded_iframe = %(<div style="position:relative;height: 0;padding-bottom:56.25%"><iframe src="https:// www.youtube.com/embed/#{matches['video_id']}?ecver=2" width="640" height="360" frameborder="0" style="position:absolute;width:100%;height: 100%;left:0" allowfullscreen></iframe></div>).gsub('{{youtube_video_id}}', youtube_video_id)
end
```

We can actually do it better by not introducing intermediate variable ``matches``:

```ruby
str = 'https://www.youtube.com/watch?v=KO8PV6tMXeo'
str.gsub(/.*v\=(\w+)/, '<div style="position:relative;height:0;padding-bottom:56.25%"><iframe src=“https://www.youtube.com/embed/\1?ecver=2" width="640" height="360" frameborder="0" style="position:absolute;width:100%;height:100%;left:0" allowfullscreen></iframe></div>')
```

So what we did was using ``gsub`` to replace entire initial youtube video URL by the iframe HTML. With the only matched group ``(\w+)``, ``gsub`` introducde the variable ``\1`` to be used in the replaced string. Hence we can use ``\1`` in the correct position in the substituted string like this ``https://www.youtube.com/embed/\1?ecver=2``.

However, note that, for the case that you have more than 2 variables to be extracted out, it is still better to use the ``named_captures`` function as it is much clearer what are the things that you get out. ``\1``, ``\2``, ... share the same problems with the earlier metioned of ``$1``, ``$2`` is that you will not know what are they about and we can end up misplacing it.

I hope you enjoy and find something useful in this article. Happy using regular expression :p.
