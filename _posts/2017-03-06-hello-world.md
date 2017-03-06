---
title:  "Hello world!"
date:   2017-03-06 12:04:34 +0200
categories: misc
tags: [misc, entrepreneurship]
---

{% highlight ruby linenos %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

bla bla text

{% highlight cpp linenos %}
#define DOCTEST_CONFIG_IMPLEMENT_WITH_MAIN
#include "doctest.h"

static int factorial(int number) { return number <= 1 ? number : factorial(number - 1) * number; }

TEST_CASE("testing the factorial function") {
    CHECK(factorial(0) == 1);
    CHECK(factorial(1) == 1);
    CHECK(factorial(2) == 2);
    CHECK(factorial(3) == 6);
    CHECK(factorial(10) == 3628800);
}
{% endhighlight %}