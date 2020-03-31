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

## One-liner Cheatsheet

### Conventions

This is a quick list of conventions that are decribed in more detail further down
in this guide or in the full [Objective-C Style Guide][objc].

1. Type all arrays, dictionaries, and sets
2. Remove everything that can be removed from the header file.  This includes: protocol adherence, unnecessarily exposed properties and functions.  Properties and functions should be defined with the most generic type possible.
3. Naming 
   1. _ before ivars. 
   2. _ before private methods.
   3. No "get" naming on properties or functions.
   4. Limit "with" and "and" in function names.
   5. Include caller as first parameter of all delegate callbacks (see every Apple example).
   6. Prefix every class
   7. Prefix every enum name.  Prefix every value with the enum name.
4. Use modern syntax.  `@[]`, `@{}`, `@()` instead of array/dictionary/number.
5. If your API don't support something, use `NSAssert`.  Crash internally instead of silently returning `nil`.

### Restrictions

This is a list of coding restrictions that lead to code that is easier to read and debug.

1. `if` statements should not contain comparisons or parentheses.  Instead store these in local `BOOL` variables with descriptive names.
2. `return` should always return a local variable.  Function calls or property lookup should first be stored to a local variable.  This makes debugging easier.  Exceptions include: returning empty arrays, dictionaries, or strings.
3. No web service (`ECF`) calls in the implementation of a `UIViewController`.  Put that into a loader/manager.
4. No just-in-time ivar creation.  Create these in your `init` when possible.
5. No over optimization.  We don't need to see `[NSArray arrayWithCapacity:]`.
6. No more than 7 ivars in `UIViewController` subclasses.
7. Never override `viewWillLayoutSubviews`, `viewDidLayoutSubviews`, `viewWillUpdateConstraints`, or `viewDidUpdateConstraints`.
8. Never use Visual Format Language for layout constraints.  You should using layout anchors like `safeAreaLayoutGuide`.
9. Never have 4 levels of indentation in a method.  Rarely have 3.
10. Never implement instance level convenience initializers.  You can have class level convenience initializers as long as they only use the public interface of the class.
11. Make liberal use of class methods, categories, and C methods in implementation files to better define the inputs and outputs and remove any unclear side effects a function has.  These tend to also be more unit testable.

## Naming cheatsheet

See the full [Objective-C Style Guide][objc] for additional details.

### Preferred

```objc

// => Avoid non-standard abbreviations
int numberOfErrors = 0;
int completedConnectionsCount = 0;
detailsViewController = [DetailsViewController new];

// => Use all capitals for acronyms and initialisms within the name
serviceURL = [NSURL URLWithString:@"https://server/path"];

// => Prefix classes, protocols, global constants, and enums with three capital letters
@protocol GTMExampleDelegate <NSObject>
@end

extern NSString *GTMExampleErrorDomain;

// => File names reflect the name of the class implementation that they contain
// GTMExample.h
@interface GTMExample : NSObject
@end

// => File names for categories should include the name of the class being extended
// UIViewController+GTMCrashReporting.h
@interface UIViewController (GTMCrashReporting)

// => Prefix categories
@property(nonatomic, setter=gtm_setUniqueIdentifier:) int gtm_uniqueIdentifier;
- (nullable NSData *)gtm_encodedState;
@end

// => Method name should read like a sentence.
// Use "with", "from", and "to" in the second and later parameter names only where necessary
- (void)addTarget:(id)target action:(SEL)action;                          // GOOD; no conjunction needed
- (CGPoint)convertPoint:(CGPoint)point fromView:(UIView *)view;           // GOOD; conjunction clarifies parameter
- (void)replaceCharactersInRange:(NSRange)aRange
            withAttributedString:(NSAttributedString *)attributedString;  // GOOD.

// => Methods that returns an object should beginning with a noun identifying the object
- (Sandwich *)sandwich;

// => Accessors should be named the same as the object it's getting
- (id)delegate;

// It should not be prefixed with the word `get`
// BAD: - (id)getDelegate;

// => local variables use camel case
int myLocalVariable;

// => instance variables use camel case and begin with an underscore
UITextField *_usernameTextField;

// => private functions begin with an underscore
- (void)_performInternalAction;

// => Prefix constants like `ClassNameConstantName` or `ClassNameEnumName`
// For swift interop, enumerated values should have names that extend the typedef name
typedef NS_ENUM(NSInteger, WRDDisplayTinge) {
  WRDDisplayTingeGreen = 1,
  WRDDisplayTingeBlue = 2,
};

// => use k for constants of static storage declared within implementation files
static const int kFileCount = 12;
static NSString *const kUserKey = @"kUserKey";
```

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

@interface GTMBook : NSObject

@property(readonly, copy, nonnull) NSString *title;
@property(readonly, copy, nullable) NSString *author;

/** The owner of the book. Setting nil resets to the default owner. */
@property(copy, null_resettable) NSString *owner;

/** Initializes a book with a title and an optional author. */
- (nonnull instancetype)initWithTitle:(nonnull NSString *)title
                               author:(nullable NSString *)author
    NS_DESIGNATED_INITIALIZER;

@end

/** Loads books from the file specified by the given path. */
NSArray<GTMBook *> *_Nullable GTMLoadBooksFromFile(NSString *_Nonnull path);
```

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