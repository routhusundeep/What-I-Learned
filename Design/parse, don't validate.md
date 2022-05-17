[Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
#function_programming #design_pattern #langsec

A wonderful idea, which I have used quite a few times, but was unable to connect the dots and figure out the underlying abstraction. It is idea primarily for type driven design.
Most of the time, this is a destination to my ever evolving software.

Read about [LANGSEC](http://langsec.org/) to know more about this philosophy

**Use a data structure that makes illegal states unrepresentable** 
Model your data using the most precise data structure you reasonably can. If ruling out a particular possibility is too hard using the encoding you are currently using, consider alternate encodings that can express the property you care about more easily. Don’t be afraid to refactor.

**Push the burden of proof upward as far as possible, but no further** 
Get your data into the most precise representation you need as quickly as you can. Ideally, this should happen at the boundary of your system, before _any_ of the data is acted upon.

In other words, write functions on the data representation you _wish_ you had, not the data representation you are given. The design process then becomes an exercise in bridging the gap, often by working from both ends until they meet somewhere in the middle. Don’t be afraid to iteratively adjust parts of the design as you go, since you may learn something new during the refactoring process!

- **Let your datatypes inform your code, don’t let your code control your datatypes**
- **Don’t be afraid to parse data in multiple passes**
- **Avoid denormalized representations of data, _especially_ if it’s mutable**
	- **Keep denormalized representations of data behind abstraction boundaries**
- **Use abstract datatypes to make validators “look like” parsers**