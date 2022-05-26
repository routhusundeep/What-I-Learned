#easy_to_read 
[Paper](https://github.com/papers-we-love/papers-we-love/blob/master/design/out-of-the-tar-pit.pdf)

This adds on top of [[No Silver Bullet]]

### Complexity
> "...we have to keep it crisp, disentangled, and simple if we refuse to be crushed by the complexities of our own making"
> - Dijkstra

> "The general problem with ambitious systems is complexity."
> "...it is important to emphasize the value of simplicity and elegance, for complexity has a way of compounding difficulties" 
> - Corbato

> "I conclude that there are two ways of constructing a software design: One way is to make it so simple that there are obviously no deficiencies, and the other way is to make it so complicated that there are no obvious deficiencies. The first method is far more difficult."
>- Hoare

Complexity is the root cause of the vast majority of problems with software today. Unreliability, late delivery, lack of security — often even poor performance in large-scale systems can all be seen as deriving ultimately from unmanageable complexity.

There are two widely-used approaches to understanding systems (or components of systems):
- Testing
- Informal Reasoning

Of the two, informal reasoning is the most important by far because there is a limit to testing. Informal reasoning will lead to fewer errors being created, whilst all that improvements in testing can do is to lead to more errors being detected.

> "Those who want really reliable software will discover that they must find means of avoiding the majority of bugs to start with"
> "Testing is hopelessly inadequate....(it) can be used very effectively to show the presence of bugs but never to show their absence"
> - Dijkstra

### Causes of Complexity
##### Complexity caused by State
The reason that many of the errors exist is that the presence of state makes programs hard to understand. It makes them complex.

> "From the complexity comes the difficulty of enumerating, much less understanding, all the possible states of the program, and from that comes the unreliability"
> - Brooks

The single biggest remaining cause of complexity in most contemporary large systems is state, and the more we can do to limit and manage state, the better.

##### Complexity caused by Control
Informal reasoning should take control/order into consideration.
Concurrency is also a branch of Control

##### Complexity caused by Code Volume
> "Many of the classic problems of developing software products derive from this essential complexity and its nonlinear increase with size"
> - Brooks

> "It has been suggested that there is some kind of law of nature telling us that the amount of intellectual effort needed grows with the square of program length. But, thank goodness, no one has been able to prove this law. And this is because it need not be true.... I tend to the assumption — up till now not disproved by experience — that by suitable application of our powers of abstraction, the intellectual effort needed to conceive or to understand a program need not grow more than proportional to program length"
> - Dijkstra

##### Other causes of complexity
- Complexity breeds complexity
	- This covers all complexity introduced as a result of not being able to clearly understand a system.
	- Duplication
- Simplicity is Hard
- Power corrupts
	- absence of language enforced guarantees

### Classical approaches to managing complexity
##### Object-Orientation

> "In a sense, object identity can be considered to be a rejection of the “relational algebra” view of the world in which two objects can only be distinguished through differing attributes"
> - Baker

The bottom line is that all forms of OOP rely on state (contained within objects) and in general all behavior is affected by this state. As a result of this, OOP suffers directly from the problems associated with state described above, and as such we believe that it does not provide an adequate foundation for avoiding complexity

Conventional imperative and object-oriented programs suffer greatly from both state-derived and control-derived complexity.

##### Functional Programming
The primary strength of functional programming is that by avoiding state (and side effects) the entire system gains the property of referential transparency — which implies that when supplied with a given set of arguments, a function will always return exactly the same result. Everything which can possibly affect the result in any way is always immediately visible in the actual parameters.  
It is this cast iron guarantee of referential transparency that obliterates one of the two crucial weaknesses of testing, as discussed above. As a result, even though the other weakness of testing remains (testing for one set of inputs says nothing at all about behavior with another set of inputs), testing does become far more effective if a system has been developed in a functional style.  
By avoiding state, functional programming also avoids all the other state-related weaknesses discussed above, so — for example — informal reasoning also becomes much more effective.

Functional programming goes a long way towards avoiding the problems of state-derived complexity. This has very significant benefits for testing (avoiding what is normally one of testing’s biggest weaknesses) as well as for reasoning.

##### Logic Programming
One of the most interesting things about logic programming is that (despite the limitations of some actual logic-based languages) it offers the tantalizing promise of the ability to escape from the complexity problems caused by control

### Accidents and Essence
**Essential Complexity** is inherent in, and the essence of, the problem (as seen by the users).  
**Accidental Complexity** is all the rest — complexity with which the development team would not have to deal in the ideal world (e.g. complexity arising from performance issues and from suboptimal language and infrastructure)

Brooks asserts that “The complexity of software is an essential property, not an accidental one”. This would suggest that the majority of the complexity that we find in contemporary large systems is of the essential type. We disagree. Complexity itself is not an inherent (or essential) property of software (it is perfectly possible to write software which is simple and yet is still software), and further, much complexity that we do see in existing software is not essential (to the problem). When it comes to accidental and essential complexity, we firmly believe that the former exists and that the goal of software engineering must be both to eliminate as much of it as possible, and to assist with the latter.

### Recommended General Approach
In fact, it is our belief that the vast majority of state (as encountered in typical contemporary systems) simply isn’t needed (in this ideal world). Because of this, and the huge complexity which state can cause, the ideal world removes all non-essential state. There is no other state at all. No caches, no stores of derived calculations of any kind. One effect of this is that all the state in the system is visible to the user of (or person testing) the system (because inputs can reasonably be expected to be visible in ways which internal cached state normally is not).

Whereas we have seen that some state is essential, control generally can be completely omitted from the ideal world and as such is considered entirely accidental.

In the ideal world, we have been able to avoid large amounts of complexity — both state and control. As a result, it is clear that a lot of complexity is accidental. This gives us hope that it may be possible to significantly reduce the complexity of real large systems. The question is — how close is it possible to get to the ideal world in the real one?


##### Formal Specification Languages
We observed that in the ideal world we would like to be able to execute the formal requirements without first having to translate them into some other language.
- Property-based approaches focus (in a declarative manner) on what is required rather than how the requirements should be achieved.
- Model-based (or State-based) approaches construct a potential model for the system (often a stateful model) and specify how that model must behave.

##### Required Accidental Complexity
In practice, we may require complexity which strictly is accidental. These reasons are:
- Performance: making use of accidental state and control can be required for efficiency
- Ease of Expression making use of accidental state can be the most natural way to express logic in some cases

##### Recommendations
- Avoid state and control where they are not absolutely and truly essential.
- Separate it from the rest of the system

This notion of restricting the power of the individual languages is an important one — the weaker the language, the more simple it is to reason about. This has something in common with the ideas behind “Domain Specific Languages” — one exception being that the domains in question are of a fairly abstract nature and combine to form a general-purpose platform.

These restrictions are absolute, and because of this provide a huge aid to understanding the different components of the system independently.
- **Essential State:** This can be seen as the foundation of the system. The specification of the required state is completely self-contained — it can make no reference to either of the other parts which must be specified. One implication of this is that changes to the essential state specification itself may require changes in both the other specifications, but changes in either of the other specifications may never require changes to the specification of essential state.
- **Essential Logic:** This is in some ways the “heart” of the system — it expresses what is sometimes termed the “business” logic. This logic expresses — in terms of the state — what must be true. It does not say anything about how, when, or why the state might change dynamically — indeed, it wouldn’t make sense for the logic to be able to change the state in any way.
- **Accidental State and Control:** This (by virtue of its accidental nature) is conceptually the least important part of the system. Changes to it can never affect the other specifications (because neither of them make any reference to any part of it), but changes to either of the others may require changes here

The rest talks about functional and relational models and their advantages. 
There is also an example system written using these paradigms.
I found this insightful, but it will be hard to summarize it, so read the paper if you want the details.

