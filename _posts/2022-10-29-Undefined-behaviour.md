---
layout: post
title: Compilers May Throw Away Your Safety Checks During Optimization!
---

Someone filed a [bug report](https://gcc.gnu.org/bugzilla/show_bug.cgi?format=multiple&id=30475) claiming that GCC has a bug how it handles signed integer under/overflow and this individual got the following response:

> signed type overflow is undefined by the C standard, use unsigned int for the addition or use -fwrapv.

The bug reporter seemingly was frustrated as GCC in previous version did not have this "issue". Here is one of the responses he wrote:

> You have GOT to be kidding?
>
> All kinds of security issues are caused by integer wraps, and you are just telling me that with gcc 4.1 and up I cannot test for them for signed data types any more?!
>
> You are missing the point here.  There HAS to be a way to get around this.  Existing software uses signed int and I can't just change it to unsigned int, but I still must be able to check for a wrap!  There does not appear to be a work around I could do in the source code either!  Do you expect me to cast it to unsigned, shift right by one, and then add or what?!
>
> PLEASE REVERT THIS CHANGE.  This will create MAJOR SECURITY ISSUES in ALL MANNER OF CODE.  I don't care if your language lawyers tell you gcc is right.  THIS WILL CAUSE PEOPLE TO GET HACKED.
>
> I found this because one check to prevent people from getting hacked failed.
>
> THIS IS NOT A JOKE.  FIX THIS!  NOW!

Finally, I want to highlight that initial responder concluded that (the thread goes on with a lot of responses, take a look for yourself):

> I am not joking, the C standard explictly says signed integer overflow is undefined behavior.

There are countless posts on bug reporting portals and on forums such as Stack Overflow about this issue. But who is correct? Let us take a concrete example of this issue at hand. Consider that we need to implement a secure and robust function for addition that avoids underflow and overflow. For our purposes we would simply return 0 to indicate that we have an overflow/underflow. C-code for this problem could look like this:

{% highlight C linenos %}
int safe_addition(int a, int b) {
  if (a > 0) {
    if (b > 0 && a + b < 0) {
      return 0; // overflow
    }
  } else {
    if (b < 0 && a + b > 0) {
      return 0; // underflow
    }
  }
  return a + b;
}
{% endhighlight %}

Compiling this in GCC 12.2 [(Godbolt)](https://godbolt.org/z/h8hWhKEhM) using O0 (i.e. no optimizations) will result in the following:
{% highlight _ %}
safe_addition(1, INT_MAX)  -> 0
safe_addition(-1, INT_MIN) -> 0
{% endhighlight %}

It works as expected, both underflow and overflow is successfully captured by our logic and we return 0 in both cases. Now using the same code but enabling compiler optimizations using O2 [(Godbolt)](https://godbolt.org/z/hqvj7xdWM), we instead get the following output:

{% highlight _ %}
safe_addition(1, INT_MAX)  -> -2147483648
safe_addition(-1, INT_MIN) -> 2147483647
{% endhighlight %}

The observed result of our function is now broken, as it no longer seems to protect us against underflow/overflow. Code like this have been found in applications such as SQLite, PostgreSQL, [Python](https://bugs.python.org/issue33632), [OpenSSL](https://git.openssl.org/?p=openssl.git;a=commit;h=a004e72b95835136d3f1ea90517f706c24c03da7), etc. (1). These checks may be very important and as such removing them might be a source of severe vulnerability, opening up the possibility for buffer overflow attacks etc.

So is the code wrong or do the compiler have a bug? The code is wrong! A compiler for a mature language like C is promising to follow a strictly defined specification and anything that is not defined by the specification will cause _undefined behavior_. A compiler is free to do whatever it wants whenever it encounters undefined behavior, it may delete your hard drive, crash, produce a correct result or produce a wrong result (or famously known in the programming language community it may also make demons fly out of your nose). In the C standard overflow and underflow is undefined behavior for signed integers. The implications of this is that the compiler is free to assume that it will never happen. So with an optimizing pass the compiler will turn our code into (GCC => 4.1):

{% highlight C linenos %}
int safe_addition(int a, int b) {
  if (a > 0) {
    if (b > 0 && false) { // a + b < 0 turned into false
      return 0; // overflow
    }
  } else {
    if (b < 0 && false) { // a + b > 0 turned into false
      return 0; // underflow
    }
  }
  return a + b;
}
{% endhighlight %}

Then with dead code elimination our code will look like:
{% highlight C linenos %}
int safe_addition(int a, int b) {
  return a + b;
}
{% endhighlight %}

From the perspective from a compiler it is sane to assume that undefined behavior will not happen and thus it is only logical to turn such expression that test for undefined behavior that themself are relying on undefined behavior to false. But note that it would have been *legal* to turn these expressions to *true*.

I am going confess that I was not aware of this particular undefined behavior issue until I saw this example (and I totally feel I should have!). I did not uncover anything here by myself, all credits goes to [Christian Cadar](https://www.doc.ic.ac.uk/~cristic), that highlighted these issues and the broader issues in programming community in a recent talk I had the pleasure to listen to. There seems to be a cognitive dissonance between application engineers and compiler engineers about the effects of undefined behavior and when it is allowed to exist. Spread the word. The initial responder to the bug report was not kidding.

(1) Dietz, Will, et al. "Understanding integer overflow in C/C++." ACM Transactions on Software Engineering and Methodology (TOSEM) 25.1 (2015): 1-29.