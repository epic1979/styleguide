# Epic Objective-C Style Guide (for PQA)

> The purpose of this document is to describe the Objective-C coding
> guidelines and practices that we see most commonly not followed.  This
> is not a fault of the developer, before this guide we have never been
> clear on these expectations.  These guidelines have evolved and been
> proven over time on other projects and teams. Projects developed by Epic
> should conform to the requirements in this guide and existing code should
> be updated as files are touched.
>
> Note that this guide is not comprehensive.  We assume that the reader
> is familiar with the full [Objective-C Style Guide][objc].


## Principles

### Optimize for the reader, not the writer

Codebases often have extended lifetimes and more time is spent reading the code
than writing it. We explicitly choose to optimize for the experience of our
average software engineer reading, maintaining, and debugging code in our
codebase rather than the ease of writing said code. For example, when something
surprising or unusual is happening in a snippet of code, leaving textual hints
for the reader is valuable.

## Naming 

Names should be as descriptive as possible, within reason. Follow standard
[Objective-C naming
rules](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html). Avoid non-standard abbreviations (including non-standard acronyms and initialisms).

```objectivec 
// GOOD:

// Good names.
int numberOfErrors = 0;
int completedConnectionsCount = 0;
tickets = [NSMutableArray new];
userInfo = [someObject object];
port = [network port];
NSDate *gAppLaunchDate;
```

```objectivec 
// AVOID:

// Names to avoid.
int w;
int nerr;
int nCompConns;
tix = [NSMutableArray new];
obj = [someObject object];
p = [network port];
```

### Prefixes

Classes and protocols should generally be named with a prefix that begins
with three capital letters.

```objectivec 
// GOOD:

/** An example error domain. */
extern NSString *GTMExampleErrorDomain;

/** Gets the default time zone. */
extern NSTimeZone *GTMGetDefaultTimeZone(void);

/** An example delegate. */
@protocol GTMExampleDelegate <NSObject>
@end

/** An example class. */
@interface GTMExample : NSObject
@end

```

### Objective-C Method Names 

Use prepositions and conjunctions like "with", "from", and "to" in the second
and later parameter names only where necessary to clarify the meaning or
behavior of the method.

```objectivec 
// GOOD:

- (void)addTarget:(id)target action:(SEL)action;                          // GOOD; no conjunction needed
- (CGPoint)convertPoint:(CGPoint)point fromView:(UIView *)view;           // GOOD; conjunction clarifies parameter
- (void)replaceCharactersInRange:(NSRange)aRange
            withAttributedString:(NSAttributedString *)attributedString;  // GOOD.
```

A method that returns an object should have a name beginning with a noun
identifying the object returned:

```objectivec 
// GOOD:

- (Sandwich *)sandwich;      // GOOD.
```

```objectivec 
// AVOID:

- (Sandwich *)makeSandwich;  // AVOID.
```

An accessor method should be named the same as the object it's getting, but it
should not be prefixed with the word `get`. For example:

```objectivec 
// GOOD:

- (id)delegate;     // GOOD.
```

```objectivec 
// AVOID:

- (id)getDelegate;  // AVOID.
```

Accessors that return the value of boolean adjectives have method names
beginning with `is`, but property names for those methods omit the `is`.

Dot notation is used only with property names, not with method names.

```objectivec 
// GOOD:

@property(nonatomic, getter=isGlorious) BOOL glorious;
- (BOOL)isGlorious;

BOOL isGood = object.glorious;      // GOOD.
BOOL isGood = [object isGlorious];  // GOOD.
```

```objectivec 
// AVOID:

BOOL isGood = object.isGlorious;    // AVOID.
```

See [Apple's Guide to Naming
Methods](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingMethods.html#//apple_ref/doc/uid/20001282-BCIGIJJF)
for more details on Objective-C naming.

### Variable Names 

Variable names typically start with a lowercase and use mixed case to delimit
words. For example: `myLocalVariable`.  Instance variable names are mixed case and
should be prefixed with an underscore, like `_myInstanceVariable`.

## Cocoa and Objective-C Features 

### Identify Designated Initializer 

Clearly identify your designated initializer using the `NS_DESIGNATED_INITIALIZER` macro.

### Override Designated Initializer 

When writing a subclass that requires an `init...` method, make sure you
override the designated initializer of the superclass. If you fail to override
the designated initializer of the superclass, your initializer may not be called
in all cases, leading to subtle and very difficult to find bugs.

### Keep the Public API Simple 

Keep your class simple; avoid "kitchen-sink" APIs. If a method doesn't need to
be public, keep it out of the public interface.

### Setters copy NSStrings 

Setters taking an `NSString` should always copy the string it accepts. This is
often also appropriate for collections like `NSArray` and `NSDictionary`.

Never just retain the string, as it may be a `NSMutableString`. This avoids the
caller changing it under you without your knowledge.

Code receiving and holding collection objects should also consider that the
passed collection may be mutable, and thus the collection could be more safely
held as a copy or mutable copy of the original.

```objectivec 
// GOOD:

@property(nonatomic, copy) NSString *name;

- (void)setZigfoos:(NSArray<Zigfoo *> *)zigfoos {
  // Ensure that we're holding an immutable collection.
  _zigfoos = [zigfoos copy];
}
```

### Use Lightweight Generics to Document Contained Types 

Every `NSArray`, `NSDictionary`, or `NSSet` reference should be declared using
lightweight generics for improved type safety and to explicitly document usage.

```objectivec 
// GOOD:

@property(nonatomic, copy) NSArray<Location *> *locations;
@property(nonatomic, copy, readonly) NSSet<NSString *> *identifiers;

NSMutableArray<MyLocation *> *mutableLocations = [otherObject.locations mutableCopy];
```

### Nullability

Interfaces can be decorated with nullability annotations to describe how the
interface should be used and how it behaves.

```objectivec 
// GOOD:

/** A class representing an owned book. */
@interface GTMBook : NSObject

/** The title of the book. */
@property(readonly, copy, nonnull) NSString *title;

/** The author of the book, if one exists. */
@property(readonly, copy, nullable) NSString *author;

/** The owner of the book. Setting nil resets to the default owner. */
@property(copy, null_resettable) NSString *owner;

/** Initializes a book with a title and an optional author. */
- (nonnull instancetype)initWithTitle:(nonnull NSString *)title
                               author:(nullable NSString *)author
    NS_DESIGNATED_INITIALIZER;

/** Returns nil because a book is expected to have a title. */
- (nullable instancetype)init;

@end

/** Loads books from the file specified by the given path. */
NSArray<GTMBook *> *_Nullable GTMLoadBooksFromFile(NSString *_Nonnull path);
```

```objectivec 
// AVOID:

NSArray<GTMBook *> *__nullable GTMLoadBooksFromTitle(NSString *__nonnull path);
```

Be careful assuming that a pointer is not null based on a non-null qualifier
because the compiler may not guarantee that the pointer is not null.

## Cocoa Patterns 

### Delegate Pattern 

Delegates, target objects, and block pointers should not be retained when doing
so would create a retain cycle.  Instead, prefer to weakly capture these references.

## Spacing and Formatting 

### Spaces vs. Tabs 

Use only spaces, and indent 4 spaces at a time. We use spaces for indentation.
Do not use tabs in your code.

You should set your editor to emit spaces when you hit the tab key, and to trim
trailing spaces on lines.

### Line Length 

The maximum line length for Objective-C files is 100 columns.

You can make violations easier to spot by enabling *Preferences > Text Editing >
Page guide at column: 100* in Xcode.

### Method Declarations and Definitions 

One space should be used between the `-` or `+` and the return type, and no
spacing in the parameter list except between parameters and any * in the parameter typing.

Methods should look like this:

```objectivec 
// GOOD:

- (void)doSomethingWithString:(NSString *)theString {
  ...
}
```

### Function Length 

Prefer small and focused functions.

Long functions and methods are occasionally appropriate, so no hard limit is
placed on function length.  a function exceeds about 40 lines, think about
whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few
months may add new behavior. This could result in bugs that are hard to find.
Keeping your functions short and simple makes it easier for other people to read
and modify your code.

When updating legacy code, consider also breaking long functions into smaller
and more manageable pieces.

[objc]: objcguide.md