# Best Practices of Rust Programming Language

## Table of Contents

1. [Slices](#slices)
2. [API tips and improvements](#api-tips)
3. [Usage Tips](#usage-tips)
4. [Code readability](#readability)


<a name="slices"></a>
## Slices
First, a little recap; a slice is a constant view over an array, and &[T] is the constant view of a Vec<T>, whereas &str is the constant view of a String (just like Path is the constant view of a PathBuf and OsStr is the constant view of an OsString). Now that you have this in mind, let's continue!
When a function expects a constant argument of type Vec or String, then always write them as follows:
```
fn some_func(v: &[u8]) {
    // some code...
}
```
Instead of:
```  
fn some_code(v: &Vec<u8>) {
    // some code
}
```
And:
```
fn some_func(s: &str) {
    // some code...
}
```  

Instead of:
```
fn some_func(s: &String) {
    // some code...
}
```

You might be wondering why this is the case. So, let's imagine your function displays your Vec as ASCII characters:
```
fn print_as_ascii(v: &[u8]) {
    for c in v {
        print!("{}", *c as char);
    }
    println!("");
}
```

<a name="api-tips"></a>
## API tips and improvements
When writing a public API (either for you or other users), a few tips can really make everyone's life easier. This is where generics kick in. Let's start with Option arguments:
 Explaining the Some function:
Generally, when a function expects an Option argument, it looks like this:
```
fn some_func(arg: Option<&str>) {
    // some code
}
```

And you call it as follows:
```
some_func(Some("ratatouille"));
some_func(None);
```
Now, what if I told you that you could get rid of the Some? Nice, right? Well, this is actually pretty easy:
```
fn some_func<'a, T: Into<Option<&'a str>>>(arg: T) {
    // some code
}
```
And you can now call it as follows:
```
some_func(Some("ratatouille")); // If you *really* like to write "Some"...
some_func("ratatouille");
some_func(None);
```
Better! However, to make users' lives easier, it'll require a bit more code for whoever's writing the function. You can't use arg as it is; you need to add an extra step. Before, you'd just do this:
```
fn some_func(arg: Option<&str>) {
    if let Some(a) = arg {
        println!("{}", a);
    } else {
        println!("nothing...");
    }
}
```
Now, you'll need to add an into the call before being able to use arg:
```
fn some_func<'a, T: Into<Option<&'a str>>>(arg: T) {
    let arg = arg.into();
    if let Some(a) = arg {
        println!("{}", a);
    } else {
        println!("nothing...");
    }
}
```
And that's it. As we said before, it doesn't require much and makes users' lives easier, so why not do it?
 ### Using the Path function
Just like the previous section, this will show you some tips to make your API more comfortable to use by auto-converting it into a Path.
So, let's take an example with a function receiving a Path as an argument:
```
use std::path::Path;

fn some_func(p: &Path) {
    // some code...
}
```

There's nothing new in here. You can call this function just like this:
some_func(Path::new("tortuga.txt"));

The annoying thing, here, is that you have to build the Path yourself before sending it to the function. This is way too annoying, but we can do better!
```
fn some_func<P: AsRef<Path>>(p: P) {
    // some code...
}
```

And that's it... You can now call the function as follows:
```
some_func(Path::new("tortuga.txt")); // If you *really* like to build the "Path" by yourself...
some_func("tortuga.txt");
```
And just like for the Into trait, you need to add one line of code in order to make it work:
```
fn some_func<P: AsRef<Path>>(p: P) {
    let p: &Path = p.as_ref();
    // some code...
}
```
And that's it! Now, as long as the given type implements AsRef<Path>, you can just send it like that. For information, here's a (non-exhaustive) list of types implementing this trait:
* OsStr / OsString
* &str / String
* Path (yes, Path implements AsRef<Path> as well!) / PathBuf
* Iter
This is already quite a lot, so you should be able to do it pretty easily!

<a name="usage-tips"></a>
## Usage Tips
### Builder pattern
A builder pattern is meant to be able to build a final object through multiple calls that can be chained. An excellent example is the OpenOptions type in the Rust standard library.
Note
It's strongly recommended you use OpenOptions when you need to play with File!
```
use std::fs::OpenOptions;

let file = OpenOptions::new()
                       .read(true)
                       .write(true)
                       .create(true)
                       .open("foo.txt");
```
To make such APIs, you have two ways:
* Playing with mutable borrows
* Playing with moves
Let's start with the mutable borrows!

### Playing with mutable borrows
The first one works just like OpenOptions:
```
struct Number(u32);

impl Number {
    fn new(nb: u32) -> Number {
        Number(nb)
    }

    fn add(&mut self, other: u32) -> &mut Number {
        self.0 += other;
        self
    }

    fn sub(&mut self, other: u32) -> &mut Number {
        self.0 -= other;
        self
    }

    fn compute(&self) -> u32 {
        self.0
    }
}
```
If you wonder about self.0, just remember that it's how you access a tuple field.
And then you can call it as follow:
```
let nb = Number::new(0).add(10).sub(5).add(12).compute();
assert_eq!(nb, 17);
```
This is the first way to do it.
Note
You'll note that you need to add an ending method so that you can transform your mutable borrow into an object (otherwise, you'll have a borrow issue).
Let's now take a look at the second way to do it!

### Playing with moves
Instead of taking &mut every time, we'll directly take the object's ownership every time:
```
struct Number(u32);

impl Number {
    fn new(nb: u32) -> Number {
        Number(nb)
    }

    fn add(mut self, other: u32) -> Number {
        self.0 += other;
        self
    }

    fn sub(mut self, other: u32) -> Number {
        self.0 -= other;
        self
    }
}
```
Then, there's no more need for the ending method:
```
let nb = Number::new(0).add(10).sub(5).add(12);
assert_eq!(nb.0, 17);
```
I generally prefer this way of doing builder patterns but it's more of a personal opinion than a thoughtful decision. Pick whichever seems to fit the best in your situation!

<a name="readability"></a>
## Code readability
We'll now talk about Rust's syntax itself. A few things can improve the code readability and are important to know. Let's start with big numbers.
### Big number formatting
It's not uncommon to see huge constant numbers in code, such as this:
```
let x = 1000000000;
```
However, this is quite difficult to read for us (human brains aren't very efficient at parsing such numbers). In Rust, you can insert _ characters into numbers without any problem:
```
let x = 1_000_000_000;
```
A lot better, right? 

### Specifying types
The Rust compiler can automatically detect the type of a variable in most cases. However, for people reading the code, it's not always obvious what a code returns. An example? Sure!
```
let x = "a 10 11 coucou 12 14".split(' ')
                              .filter_map(|e| e.parse::<u32>().ok())
                              .filter(|x| x % 2 == 0)
                              .map(|s| format!("{}", s))
                              .collect::<Vec<_>>()
                              .join("::");
```

After reading the code carefully, you'll guess that x is a String. However, you needed to read all those closures to get it and even then, are you really sure of the type?


In such cases, it's strongly recommended to just add the type annotation:
```
let x: String = "a 10 11 coucou 12 14".split(' ')
                                      .filter_map(|e| e.parse::<u32>().ok())
                                      .filter(|x| x % 2 == 0)
                                      .map(|s| format!("{}", s))
                                      .collect::<Vec<_>>()
                                      .join("::");

```
It doesn't cost much and allows readers (including you) to go through the code so much faster.
### Matching
It's common to use pattern matching through match blocks in Rust. However, it's often a better solution to use if let conditions. Let's take a simple example:
```
enum SomeEnum {
     Ok,
     Err,
     Unknown,
}
```
Now let's say you want to perform an action only when you get Ok. With a match, you would do this:
```
let x = SomeEnum::Err;

match x {
    SomeEnum::Ok => {
        // Huge code doing a lot of things...
    }
    _ => {}
}
```
Not really an issue, right? Now let's see it with an if let:
```
let x = SomeEnum::Err;

if let SomeEnum::Ok = x {
    // Huge code doing a lot of things...
}
```
And that's it. It basically makes the code a little shorter, while improving readability a lot. Whenever you just need to get one value, it's often a better solution to use if let instead of match.
