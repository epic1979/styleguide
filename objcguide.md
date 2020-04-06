# Epic Objective-C Style Guide

> Objective-C is a dynamic, object-oriented extension of C. It's designed to be
> easy to use and read, while enabling sophisticated object-oriented design. It
> is the primary development language for applications on OS X and on iOS.
>
> Apple has already written a very good, and widely accepted, [Cocoa Coding
> Guidelines](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html)
> for Objective-C. Please read it in addition to this guide.
>
>
> The purpose of this document is to describe the Objective-C (and
> Objective-C++) coding guidelines and practices that should be used for iOS.
> These guidelines have evolved and been proven over time on other
> projects and teams.
> Projects developed by Epic conform to the requirements in this guide.
>
> Note that this guide is not an Objective-C tutorial. We assume that the reader
> is familiar with the language. If you are new to Objective-C or need a
> refresher, please read [Programming with
> Objective-C](https://developer.apple.com/library/mac/#documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Introduction/Introduction.html).

## Principles

### Optimize for the reader, not the writer

Codebases often have extended lifetimes and more time is spent reading the code
than writing it. We explicitly choose to optimize for the experience of our
average software engineer reading, maintaining, and debugging code in our
codebase rather than the ease of writing said code. For example, when something
surprising or unusual is happening in a snippet of code, leaving textual hints
for the reader is valuable.

### Be consistent

When the style guide allows multiple options it is preferable to pick one option
over mixed usage of multiple options. Using one style consistently throughout a
codebase lets engineers focus on other (more important) issues. Consistency also
enables better automation because consistent code allows more efficient
development and operation of tools that format or refactor code. In many cases,
rules that are attributed to "Be Consistent" boil down to "Just pick one and
stop worrying about it"; the potential value of allowing flexibility on these
points is outweighed by the cost of having people argue over them.

### Be consistent with Apple SDKs

Consistency with the way Apple SDKs use Objective-C has value for the same
reasons as consistency within our code base. If an Objective-C feature solves a
problem that's an argument for using it. However, sometimes language features
and idioms are flawed, or were just designed with assumptions that are not
universal. In those cases it is appropriate to constrain or ban language
features or idioms.

### Style rules should pull their weight

The benefit of a style rule must be large enough to justify asking engineers to
remember it. The benefit is measured relative to the codebase we would get
without the rule, so a rule against a very harmful practice may still have a
small benefit if people are unlikely to do it anyway. This principle mostly
explains the rules we don’t have, rather than the rules we do.

## Example

They say an example is worth a thousand words, so let's start off with an
example that should give you a feel for the style, spacing, naming, and so on.

```objectivec

/*!
 * A sample header demonstrating good Objective-C style. Comments must
 * be adjacent to the object they're documenting.
 */
@import Foundation;
@import UIKit;

NS_ASSUME_NONNULL_BEGIN

/** Ephemeral view events */
typedef NS_ENUM(NSInteger, WRDMessageDetailViewControllerAction) {
    WRDMessageDetailViewControllerActionEdit,
    WRDMessageDetailViewControllerActionCancel,
    WRDMessageDetailViewControllerActionSave,
};  // events the view controller may initiate

typedef NSString *WRDMessageDetailInputIdentifier;

@protocol WRDMessageDetailViewControllerDelegate;

//________________________________________________________________________________________________________
// The data model for displaying details

/** Immutable detail model */

@interface WRDMessageDetailModel : NSObject <NSCopying, NSMutableCopying>

- (instancetype)initWithHtml:(NSString *)html
                     patient:(WRDPatient *)patient
                  inputTexts:(NSDictionary<NSString *, NSString *> *)inputTexts NS_DESIGNATED_INITIALIZER;

/** html is displayed to the user as the primary content for the details view */
@property (nonatomic, readonly, copy) NSString *html;

/** patient identifiers, including name, MRN, and photo, are shown to the user to
 * ensure documentation occurs on the correct patient
 */
@property (nonatomic, readonly, copy) WRDPatient *patient;

/** native input fields are placed over their respective html elements to support
 * speech to text input natively.  Use this to configure the initial text input in
 * each field.  Each of the texts is copied.
 */
@property (nonatomic, readonly, copy) NSDictionary <WRDMessageDetailInputIdentifier, NSString *> *inputTexts;

@end

/** Mutable detail model */

@interface WRDMutableMessageDetailModel : WRDMessageDetailModel

@property (nonatomic, readwrite, copy) NSString *html;
@property (nonatomic, readwrite, copy) WRDPatient *patient;
@property (nonatomic, readwrite, copy) NSMutableDictionary <WRDMessageDetailInputIdentifier, NSString *> *inputTexts;

@end

//________________________________________________________________________________________________________

@interface WRDMessageDetailViewController : UIViewController

/** The object whose value is being presented in the view. */
@property (nonatomic, copy) WRDMessageDetailModel *representedObject;

@property (nonatomic, weak) id<WRDMessageDetailViewControllerDelegate> delegate;

/** Set this to NO to disable all actions available in the toolbar and navigationBar */
- (void)setBarActionsEnabled:(BOOL)enabled;

@end

//________________________________________________________________________________________________________
// respond to the user interaction with the view.

@protocol WRDMessageDetailViewControllerDelegate <NSObject>

/*! The caller is always passed as the first parameter in delegate methods
 */
- (void)detailController:(WRDMessageDetailViewController *)controller initiatedAction:(WRDMessageDetailViewControllerAction)action;

@end

NS_ASSUME_NONNULL_END
```

An example source file.

```objectivec
#import "WRDMessageDetailViewController.h"
@import SpeechToText;

/*! Protocol adherence in implementation file if unnecessary to expose externally.
 */
@interface WRDMessageDetailViewController () <WRDInputControllerDelegate, ATCResourceRequestor>

// readonly casted reference to the view controller's view.
@property (nonatomic, readonly) WRDMessageDetailView *detailView;

@end

@implementation WRDMessageDetailViewController {
  // store references to native input controllers
  NSMutableDictionary <WRDMessageDetailInputIdentifier, IBDLAnchorInputController*> *_inputControllers;

  // The patient header view
  WRDPatientViewController *_patientViewController;

  WRDMutableMessageDetailModel *_representedObject;

  struct {
    BOOL waitingToSave;
    BOOL audioResourceRequestFailed;
  } _flags;
}

/*! Lifecycle methods go at the top of an implementation file, with dealloc and init methods first
 */

/*
 * MARK: Lifecycle methods
 */
- (void)dealloc {
  // Give up the microphone if we still have it. If we don't, we'll be ignored anyway
  [ATCResourceCoordinator.sharedCoordinator requestor:self resignResourceType:ATCResourceTypeAudioInput];
}

- (instancetype)initWithNibName:(nullable NSString *)nibNameOrNil bundle:(nullable NSBundle *)nibBundleOrNil {
  /*! Always override superclass's designated initializer
   */
  self = [super init];
  if (self) {
    /*! Initialize non-view instance variables in the initializer
     */
    _patientViewController = [IBPatientViewController new];
    _inputControllers = [NSMutableDictionary new];

    self.automaticallyAdjustsScrollViewInsets = NO;

    self.navigationItem.rightBarButtonItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemSave target:self action:@selector(_saveTapped)];
    self.navigationItem.leftBarButtonItem = [[UIBarButtonItem alloc] initWithBarButtonSystemItem:UIBarButtonSystemItemCancel target:self action:@selector(_cancelTapped)];
  }

  return self;
}

- (void)loadView {
  /*! Most of the configuration goes in the viewDidLoad function to better support changing
   * how the view is initialized, for example from a nib or SwiftUI
   */
  self.view = [WRDMessageDetailView new];
}

- (void)viewDidLoad {
  [super viewDidLoad];

  // A side effect of configuring the view for the representedObject is that
  // _patientViewController's view gets added to our view. After this we need to
  // remember to call -didMoveToParentViewController:
  [self addChildViewController:_patientViewController];
  [self _configureViewForRepresentedObject];
  [_patientViewController didMoveToParentViewController:self];
}

- (void)viewWillDisappear:(BOOL)animated {
  [super viewWillDisappear:animated];

  if(self.isEditing) {
    [self.view endEditing:YES];
    [self _trySave];
  }

  [ATCResourceCoordinator.sharedCoordinator requestor:self resignResourceType:ATCResourceTypeAudioInput];
}

- (void)viewDidAppear:(BOOL)animated {
  [super viewDidAppear:animated];

  /*! epicConnection is not referenced until it is safe to do so in the view appearance methods
   */
  if (self.epicConnection.serverInfo.readOnly) {
    [self _disableAllAnchorControllers];
    return;
  }

  /*! Logically related code is moved into its own, well-named function to
   *  make it easier for the next developer to understand what the individual
   *  logical components of the current function are.
   */
  [self _requestAudioResource];
}

/* Overridden getter for property */
- (WRDMessageDetailView *)messageDetailView {
  // use viewIfLoaded to safeguard against accidently loading the view too soon
  return (id)self.viewIfLoaded;
}

/*
 * MARK: REPRESENTED OBJECT
 */
- (void)setRepresentedObject:(WRDMessageDetailModel *)representedObject {
  
  /*! Potentially mutable input are always copied and declared with the copy attribute in
   * the property definition in the header even if it isn't functionally necessary
   */
  _representedObject = [representedObject mutableCopy];

  [self _configureViewForRepresentedObject];
  [self _requestAudioResource];
}

-(void)_configureViewForRepresentedObject {

  // only configure view if it is loaded and there exists a represented object
  // remember to call this from both the representedObject setter and viewDidLoad
  if(!self.isViewLoaded || !_representedObject) {
    return;
  }

  // update patient controller
  _patientViewController.patient = _representedObject.patient;

  // === update input controllers ===

  // cache off existing inputControllers and add back if identifier still exists
  NSMutableDictionary *currentInputControllers = [_inputControllers mutableCopy];
  [_inputControllers removeAllObjects];

  // set of views to be passed to self.messageDetailView for the native input controllers
  NSMutableDictionary <WRDMessageDetailInputIdentifier, UIView *> *anchorViews = [NSMutableDictionary new];

  // create or reuse controllers and views for the provided input texts
  for (WRDMessageDetailInputIdentifier identifier in _representedObject.inputTexts) {
    WRDInputController *controller = currentInputControllers[identifier];

    // if a controller doesn't exist for this identifier, create one
    if(!controller) {
      controller = [WRDInputController new];
      controller.delegate = self;
      [self addChildViewController:controller];
      [controller didMoveToParentViewController:self];
    }

    // configure the controller for the current _representedObject
    controller.representedObject = _representedObject.inputTexts[identifier];

    // store the controller and add to new set of views
    _inputControllers[identifier] = controller;
    anchorViews[identifier] = controller.view;
  }

  // update the view by creating a new view model
  WRDMessageDetailViewModel *viewModel = [IBDLMessageDetailViewModel new];
  viewModel.html = _representedObject.html;
  viewModel.headerView = _patientViewController.view;
  viewModel.anchorViews = anchorViews;

  // set all update data in a single entry point by setting the view's representedObject property
  self.messageDetailView.representedObject = viewModel;
}

//...

@end
```

## Naming

Names should be as descriptive as possible, within reason. Follow standard
[Objective-C naming
rules](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/CodingGuidelines/CodingGuidelines.html).

Avoid non-standard abbreviations (including non-standard acronyms and
initialisms). Don't worry about saving horizontal space as it is far more
important to make your code immediately understandable by a new reader. For
example:

```objectivec
// GOOD:

int numberOfErrors = 0;
int completedConnectionsCount = 0;
tickets = [NSMutableArray new];
detailsViewController = [DetailsViewController new];
```

```objectivec
// AVOID:

int numErrors;
int nCompConns;
tix = [[NSMutableArray alloc] init];
detailsVC = [[DetailsViewController alloc] init];
```

Any class, category, method, function, or variable name should use all capitals
for acronyms and
[initialisms](https://en.wikipedia.org/wiki/Initialism)
within the name. This follows Apple's standard of using all capitals within a
name for acronyms such as URL, ID, TIFF, and EXIF.

### Prefixes

Prefixes are commonly required in Objective-C to avoid naming collisions in a
global namespace. Classes, protocols, and enums should generally be named with
a prefix that begins with three capital letters.  This is because Apple reserves
two-letter prefixes—see
[Conventions in Programming with Objective-C](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ProgrammingWithObjectiveC/Conventions/Conventions.html).

```objectivec
// GOOD:

extern NSString *GTMExampleErrorDomain;
extern NSTimeZone *GTMGetDefaultTimeZone(void);

@protocol GTMExampleDelegate <NSObject>
@end

@interface GTMExample : NSObject
@end

```

### Class Names

Class names (along with category and protocol names) should start as uppercase
and use mixed case to delimit words. Classes and protocols in code shared across
multiple files (i.e. outside an implementation file) must have an appropriate
[prefix](#prefixes) (e.g. GTMSendMessage).

### Category Naming

Category names should start with an appropriate [prefix](#prefixes) identifying
the category as part of a project.

Category source file names should begin with the class being extended followed
by a plus sign and the name of the category, e.g., `NSString+GTMParsing.h`.
Methods in a category should be prefixed with a lowercase version of the prefix
used for the category name followed by an underscore (e.g.,
`gtm_myCategoryMethodOnAString:`) in order to prevent collisions in
Objective-C's global namespace.

There should be a single space between the class name and the opening
parenthesis of the category.

```objectivec
// GOOD:

// UIViewController+GTMCrashReporting.h
@interface UIViewController (GTMCrashReporting)
@property(nonatomic, setter=gtm_setUniqueIdentifier:) int gtm_uniqueIdentifier;
- (nullable NSData *)gtm_encodedState;
@end
```

It is recommended to use categories in an implementation file to better separate
logic and make the code more readable.  In this case categories may omit name
prefixes and method name prefixes.

```objectivec
// GOOD:

/** This category extends a class that is not shared with other projects. */
@interface XYZDataObject (Storage)
- (NSString *)storageIdentifier;
@end
```

### Objective-C Method Names

Method and parameter names typically start as lowercase and then use mixed case.

Proper capitalization should be respected, including at the beginning of names.

```objectivec
// GOOD:

+ (NSURL *)URLWithString:(NSString *)URLString;
```

The method name should read like a sentence if possible, meaning you should
choose parameter names that flow with the method name. Objective-C method names
tend to be very long, but this has the benefit that a block of code can almost
read like prose, thus rendering many implementation comments unnecessary.

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

```objectivec
// GOOD:

@property(nonatomic, getter=isGlorious) BOOL glorious;

BOOL isGood = object.isGlorious;      // GOOD.
```

See [Apple's Guide to Naming
Methods](https://developer.apple.com/library/mac/documentation/Cocoa/Conceptual/CodingGuidelines/Articles/NamingMethods.html#//apple_ref/doc/uid/20001282-BCIGIJJF)
for more details on Objective-C naming.

### Function Names

Function names should start with a capital letter and have a capital letter for
each new word (a.k.a. "[camel case](https://en.wikipedia.org/wiki/Camel_case)"
or "Pascal case").  Because Objective-C does not provide namespacing, non-static functions should
have a [prefix](#prefixes) that minimizes the chance of a name collision.

```objectivec
// GOOD:

extern NSTimeZone *GTMGetDefaultTimeZone(void);
extern NSString *GTMGetURLScheme(NSURL *URL);
```

### Variable Names

Variable names typically start with a lowercase and use mixed case to delimit
words. For example: `myLocalVariable`.

#### Instance Variables

Instance variable names are mixed case and should be prefixed with an
underscore, like `_usernameTextField`.

#### Constants

Constant symbols (const global and static variables and constants created
with #define) should use mixed case to delimit words.  Global and file scope
constants should have an appropriate [prefix](#prefixes).

```objectivec
// GOOD:

extern NSString *const GTLServiceErrorDomain;
```

Because Objective-C does not provide namespacing, constants with external
linkage should have a prefix that minimizes the chance of a name collision,
typically like `ClassNameConstantName` or `ClassNameEnumName`.

For interoperability with Swift code, enumerated values should have names that
extend the typedef name:

```objectivec
// GOOD:

typedef NS_ENUM(NSInteger, WRDDisplayTinge) {
  WRDDisplayTingeGreen = 1,
  WRDDisplayTingeBlue = 2,
};
```

A lowercase k can be used as a standalone prefix for constants of static storage
duration declared within implementation files:

```objectivec
// GOOD:

static const int kFileCount = 12;
static NSString *const kUserKey = @"kUserKey";
```

### File Names

File names should reflect the name of the class implementation that they
contain—including case.

File names for categories should include the name of the class being extended,
like UITextView+GTMAutocomplete.h or NSString+GTMUtils.h.

## Types and Declarations

### Method Declarations

As shown in the [example](#Example), the recommended order for declarations
in an `@interface` declaration are: initializers, properties, and then
instance methods. Class methods should usually go in a convenience category
and should begin with any convenience constructors.

### Unsigned Integers

Avoid unsigned integers except when matching types used by system interfaces.

Subtle errors crop up when doing math or counting down to zero using unsigned
integers. Rely only on signed integers in math expressions except when matching
NSUInteger in system interfaces.

```objectivec
// GOOD:

NSUInteger numberOfObjects = array.count;
for (NSInteger counter = numberOfObjects - 1; counter > 0; --counter)
```

```objectivec
// AVOID:

for (NSUInteger counter = numberOfObjects - 1; counter > 0; --counter)  // AVOID.
```

## Comments

Comments are absolutely vital to keeping our code readable. The following rules
describe what you should comment and where. But remember: while comments are
important, the best code is self-documenting. Giving sensible names to types and
variables is much better than using obscure names and then trying to explain
them through comments.

Pay attention to punctuation, spelling, and grammar; it is easier to read
well-written comments than badly written ones.

Comments should be as readable as narrative text, with proper capitalization and
punctuation. In many cases, complete sentences are more readable than sentence
fragments. Comments at the end of a line of code should be avoided because it can cause
difficulties when diffing file changes during code review.
When writing your comments, write for your audience: the next contributor who will need to understand your code. Be generous—the next one may be you!

### Declaration Comments

Every non-trivial interface, public and private, should have an accompanying
comment describing its purpose and how it fits into the larger picture.

```objectivec
// GOOD:

/**
 * A delegate for NSApplication to handle notifications about app
 * launch and shutdown. Owned by the main app controller.
 */
@interface MyAppDelegate : NSObject {
  /**
   * The background task in progress, if any. This is initialized
   * to the value UIBackgroundTaskInvalid.
   */
  UIBackgroundTaskIdentifier _backgroundTaskID;
}

/** The factory that creates and manages fetchers for the app. */
@property(nonatomic) GTMSessionFetcherService *fetcherService;

@end
```

Doxygen-style comments are encouraged for interfaces as they are parsed by Xcode
to display formatted documentation.

Each method should have a comment explaining its function, arguments,
return value, thread or queue assumptions, and any side effects.
Documentation comments should be in the header for public methods, or
immediately preceding the method for non-trivial private methods.

Use descriptive form ("Opens the file") rather than imperative form ("Open the
file") for method and function comments. The comment describes the function; it
does not tell the function what to do.

Any sentinel values for properties and ivars, such as `NULL` or `-1`, should be
documented in comments.

Declaration comments explain how a method or function is used. Comments
explaining how a method or function is implemented should be with the
implementation rather than with the declaration.

### Implementation Comments

Provide comments explaining tricky, subtle, or complicated sections of code.

```objectivec
// GOOD:

// Set the property to nil before invoking the completion handler to
// avoid the risk of reentrancy leading to the callback being
// invoked again.
CompletionHandler handler = self.completionHandler;
self.completionHandler = nil;
handler();
```

When useful, also provide comments about implementation approaches that were
considered or abandoned.

### Disambiguating Symbols

Where needed to avoid ambiguity, use backticks or vertical bars to quote
variable names and symbols in comments in preference to using quotation marks
or naming the symbols inline.

In Doxygen-style comments, prefer demarcating symbols with a monospace text
command, such as `@c`.

Demarcation helps provide clarity when a symbol is a common word that might make
the sentence read like it was poorly constructed. A common example is the symbol
`count`:

```objectivec
// GOOD:

// Sometimes `count` will be less than zero.
```

or when quoting something which already contains quotes

```objectivec
// GOOD:

// Remember to call `StringWithoutSpaces("foo bar baz")`
```

Backticks or vertical bars are not needed when a symbol is self-apparent.

```objectivec
// GOOD:

// This class serves as a delegate to GTMDepthCharge.
```

Doxygen formatting is also suitable for identifying symbols.

```objectivec
// GOOD:

/** @param maximum The highest value for @c count. */
```

## C Language Features

### Macros

Avoid macros, especially where `const` variables, enums, XCode snippets, or C
functions may be used instead. Macros make the code you see different from the
code the compiler sees. Modern C renders traditional uses of macros for constants
and utility functions unnecessary.

### Nonstandard Extensions

Nonstandard extensions to C/Objective-C may not be used unless otherwise
specified.

Compilers support various extensions that are not part of standard C. Examples
include compound statement expressions (e.g. `foo = ({ int x; Bar(&x); x })`).

`__attribute__` is an approved exception, as it is used in Objective-C API
specifications.

The binary form of the conditional operator, `A ?: B`, is an approved exception.

## Cocoa and Objective-C Features

### Identify Designated Initializer

Clearly identify your designated initializer.

It is important for those who might be subclassing your class that the
designated initializer be clearly identified. That way, they only need to
override a single initializer (of potentially several) to guarantee the
initializer of their subclass is called. It also helps those debugging your
class in the future understand the flow of initialization code if they need to
step through it. Identify the designated initializer using the
`NS_DESIGNATED_INITIALIZER` macro and mark unsupported initializers with `NS_UNAVAILABLE`.

### Override Designated Initializer

When writing a subclass that requires an `init...` method, make sure you
override the designated initializer of the superclass. If you fail to override
the designated initializer of the superclass, your initializer may not be called
in all cases, leading to subtle and very difficult to find bugs.

### Overridden NSObject Method Placement

Put overridden methods of NSObject at the top of an `@implementation`.

This commonly applies to (but is not limited to) the `init...`, `copyWithZone:`,
and `dealloc` methods. The `init...` methods should be grouped together,
followed by other typical `NSObject` methods such as `description`, `isEqual:`,
and `hash`.

### Initialization

Don't initialize instance variables to `0` or `nil` in the `init` method; doing
so is redundant. All instance variables for a newly allocated object are [initialized
to](https://developer.apple.com/library/mac/documentation/General/Conceptual/CocoaEncyclopedia/ObjectAllocation/ObjectAllocation.html) `0` (except for isa).

### Keep the Public API Simple

Keep your class simple; avoid "kitchen-sink" APIs. If a method doesn't need to
be public, keep it out of the public interface.

Unlike C++, Objective-C doesn't differentiate between public and private
methods; any message may be sent to an object. As a result, avoid placing
methods in the public API unless they are actually expected to be used by a
consumer of the class. This helps reduce the likelihood they'll be called when
you're not expecting it. This includes methods that are being overridden from
the parent class.

Since internal methods are not really private, it's easy to accidentally
override a superclass's "private" method, thus making a very difficult bug to
squash. In general, private methods should have a fairly unique name that will
prevent subclasses from unintentionally overriding them.

### #import and #include

`#import` Objective-C and Objective-C++ headers. It is almost never appropriate to use `#include`.

### Use Umbrella Headers for System Frameworks

Import umbrella headers for system frameworks and system libraries rather than
include individual files.

While it may seem tempting to include individual system headers from a framework
such as Cocoa or Foundation, in fact it's less work on the compiler if you
include the top-level root framework. The root framework is generally
pre-compiled and can be loaded much more quickly. In addition, remember to use
`@import` for Objective-C frameworks.

```objectivec
// GOOD:

@import UIKit;
```

```objectivec
// AVOID:

#import <Foundation/NSArray.h>
#import <Foundation/NSString.h>

#import <Foundation/Foundation.h>     // AVOID.
...
```

### Avoid Messaging the Current Object Within Initializers and `-dealloc`

Code in initializers and `-dealloc` should avoid invoking instance methods.
Whenever practical, directly assign to and release ivars in initializers
and `-dealloc`, rather than rely on accessors.

```objectivec
// GOOD:

- (instancetype)init {
  self = [super init];
  if (self) {
    _bar = 23;  // GOOD.
  }
  return self;
}
```

Beware of factoring common initialization code into helper methods:

- Methods can be overridden in subclasses, either deliberately, or accidentally due to naming collisions.
- When editing a helper method, it may not be obvious that the code is being run from an initializer.

```objectivec
// AVOID:

- (instancetype)init {
  self = [super init];
  if (self) {
    self.bar = 23;  // AVOID.
    [self sharedMethod];  // AVOID. Fragile to subclassing or future extension.
  }
  return self;
}
```

```objectivec
// GOOD:

- (void)dealloc {
  [_notifier removeObserver:self];  // GOOD.
}
```

```objectivec
// AVOID:

- (void)dealloc {
  [self removeNotifications];  // AVOID.
}
```

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

All projects compiling on Xcode 7 or newer versions should make use of the
Objective-C lightweight generics notation to type contained objects.

Every `NSArray`, `NSDictionary`, or `NSSet` reference should be declared using
lightweight generics for improved type safety and to explicitly document usage.
Always use the most descriptive common superclass or protocol available to avoid
leaking unnecessary implementation details.

```objectivec
// GOOD:

@property(nonatomic, copy) NSArray<Location *> *locations;
@property(nonatomic, copy, readonly) NSSet<NSString *> *identifiers;

NSMutableArray<MyLocation *> *mutableLocations = [otherObject.locations mutableCopy];
```

If the fully-annotated types become complex, consider using a typedef to
preserve readability.

```objectivec
// GOOD:

typedef NSSet<NSDictionary<NSString *, NSDate *> *> TimeZoneMappingSet;
TimeZoneMappingSet *timeZoneMappings = [TimeZoneMappingSet setWithObjects:...];
```

### `nil` Checks

Avoid `nil` pointer checks that exist only to prevent sending messages to `nil`.
Sending a message to `nil` [reliably
returns](http://www.sealiesoftware.com/blog/archive/2012/2/29/objc_explain_return_value_of_message_to_nil.html)
`nil` as a pointer, zero as an integer or floating-point value, structs
initialized to `0`, and `_Complex` values equal to `{0, 0}`.

```objectivec
// AVOID:

if (dataSource) {  // AVOID.
  [dataSource moveItemAtIndex:1 toIndex:0];
}
```

```objectivec
// GOOD:

[dataSource moveItemAtIndex:1 toIndex:0];  // GOOD.
```

### Nullability

Interfaces can be decorated with nullability annotations to describe how the
interface should be used and how it behaves. Use of nullability regions (e.g.,
`NS_ASSUME_NONNULL_BEGIN` and `NS_ASSUME_NONNULL_END`) and explicit nullability
annotations are both accepted. Prefer using the `_Nullable` and `_Nonnull`
keywords over the `__nullable` and `__nonnull` keywords. For Objective-C methods
and properties prefer using the context-sensitive, non-underscored keywords,
e.g., `nonnull` and `nullable`.

```objectivec
// GOOD:

/** A class representing an owned book. */
@interface GTMBook : NSObject

@property(readonly, copy, nonnull) NSString *title;
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

### BOOL Pitfalls

Be careful when converting general integral values to `BOOL`. Avoid comparing
directly with `YES`.

Common mistakes include casting or converting an array's size, a pointer value,
or the result of a bitwise logic operation to a `BOOL` that could, depending on
the value of the last byte of the integer value, still result in a `NO` value.
When converting a general integral value to a `BOOL`, use ternary operators to
return a `YES` or `NO` value.

Using logical operators (`&&`, `||` and `!`) with `BOOL` is also valid and will
return values that can be safely converted to `BOOL` without the need for a
ternary operator.

```objectivec
// AVOID:

- (BOOL)isBold {
  return [self fontTraits] & NSFontBoldTrait;  // AVOID.
}
- (BOOL)isValid {
  return [self stringValue];  // AVOID.
}
```

```objectivec
// GOOD:

- (BOOL)isBold {
  return ([self fontTraits] & NSFontBoldTrait) ? YES : NO;
}
- (BOOL)isValid {
  return [self stringValue] != nil;
}
- (BOOL)isEnabled {
  return [self isValid] && [self isBold];
}
```

Also, don't directly compare `BOOL` variables directly with `YES`. Not only is
it harder to read but the first point above demonstrates that return values may
not always be what you expect.

```objectivec
// AVOID:

BOOL great = [foo isGreat];
if (great == YES) {  // AVOID.
  // ...be great!
}
```

```objectivec
// GOOD:

BOOL great = [foo isGreat];
if (great) {         // GOOD.
  // ...be great!
}
```

### Interfaces Without Instance Variables

Omit the empty set of braces on interfaces that do not declare any instance
variables.

```objectivec
// GOOD:

@interface MyClass : NSObject

@end
```

```objectivec
// AVOID:

@interface MyClass : NSObject {
}

@end
```

## Cocoa Patterns

### Delegate Pattern

Delegates, target objects, and block pointers should not be retained when doing
so would create a retain cycle.

To avoid causing a retain cycle, a delegate or target pointer should be released
as soon as it is clear there will no longer be a need to message the object.

If there is no clear time at which the delegate or target pointer is no longer
needed, the pointer should only be retained weakly.

Block pointers cannot be retained weakly. To avoid causing retain cycles in the
client code, block pointers should be used for callbacks only where they can be
explicitly released after they have been called or once they are no longer
needed. Otherwise, callbacks should be done via weak delegate or target
pointers.

## Spacing and Formatting

### Spaces vs. Tabs

Use only spaces, and indent 2 spaces at a time. We use spaces for indentation.
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

If a method declaration does not fit on a single line, put each parameter on its
own line. All lines except the first should be indented at least four spaces.
Colons before parameters should be aligned on all lines. If the colon before the
parameter on the first line of a method declaration is positioned such that
colon alignment would cause indentation on a subsequent line to be less than
four spaces, then colon alignment is only required for all lines except the
first.

```objectivec
// GOOD:

- (void)doSomethingWithFoo:(GTMFoo *)theFoo
                      rect:(NSRect)theRect
                  interval:(float)theInterval {
  ...
}

- (void)shortKeyword:(GTMFoo *)theFoo
            longerKeyword:(NSRect)theRect
    someEvenLongerKeyword:(float)theInterval
                    error:(NSError **)theError {
  ...
}

- (id<UIAdaptivePresentationControllerDelegate>)
    adaptivePresentationControllerDelegateForViewController:(UIViewController *)viewController;

- (void)presentWithAdaptivePresentationControllerDelegate:
    (id<UIAdaptivePresentationControllerDelegate>)delegate;
```

### Function Declarations and Definitions

Prefer putting the return type on the same line as the function name and append
all parameters on the same line if they will fit. Wrap parameter lists which do
not fit on a single line as you would wrap arguments in a [function
call](#Function_Calls).

```objectivec
// GOOD:

NSString *GTMVersionString(int majorVersion, int minorVersion) {
  ...
}

void GTMSerializeDictionaryToFileOnDispatchQueue(
    NSDictionary<NSString *, NSString *> *dictionary,
    NSString *filename,
    dispatch_queue_t queue) {
  ...
}
```

Function declarations and definitions should also satisfy the following
conditions:

- The opening parenthesis must always be on the same line as the function
    name.
- If you cannot fit the return type and the function name on a single line,
    break between them and do not indent the function name.
- There should never be a space before the opening parenthesis.
- There should never be a space between function parentheses and parameters.
- The open curly brace is always on the end of the last line of the function
    declaration, not the start of the next line.
- The close curly brace is either on the last line by itself or on the same
    line as the open curly brace.
- There should be a space between the close parenthesis and the open curly
    brace.
- All parameters should be aligned if possible.
- Function scopes should be indented 2 spaces.
- Wrapped parameters should have a 4 space indent.

### Conditionals

Include a space after `if`, `while`, `for`, and `switch`, and around comparison
operators.

```objectivec
// GOOD:

for (int i = 0; i < 5; ++i) {
}

while (test) {};
```

If an `if` clause has an `else` clause, both clauses should use braces.

```objectivec
// GOOD:

if (hasBaz) {
  foo();
} else {  // The else goes on the same line as the closing brace.
  bar();
}
```

```objectivec
// AVOID:

if (hasBaz) foo();
else bar();        // AVOID.

if (hasBaz) {
  foo();
} else bar();      // AVOID.
```

Intentional fall-through to the next case should be documented with a comment
unless the case has no intervening code before the next case.

```objectivec
// GOOD:

switch (i) {
  case 1:
    ...
    break;
  case 2:
    j++;
    // Falls through.
  case 3: {
    int k;
    ...
    break;
  }
  case 4:
  case 5:
  case 6: break;
}
```

### Expressions

Use a space around binary operators and assignments. Omit a space for a unary
operator. Do not add spaces inside parentheses.

```objectivec
// GOOD:

x = 0;
v = w * x + y / z;
v = -y * (x + z);
```

Factors in an expression may omit spaces.

```objectivec
// GOOD:

v = w*x + y/z;
```

### Method Invocations

Method invocations should be formatted much like method declarations. As with
declarations and definitions, when the first keyword is shorter than the others,
indent the later lines by at least four spaces, maintaining colon alignment.

```objectivec
// GOOD:

[myObject doFooWith:arg1 name:arg2 error:arg3];

[myObject doFooWith:arg1
               name:arg2
              error:arg3];

[myObj short:arg1
          longKeyword:arg2
    evenLongerKeyword:arg3
                error:arg4];
```

Don't use any of these styles:

```objectivec
// AVOID:

[myObject doFooWith:arg1 name:arg2  // some lines with >1 arg
              error:arg3];

[myObject doFooWith:arg1
               name:arg2 error:arg3];

[myObject doFooWith:arg1
          name:arg2  // aligning keywords instead of colons
          error:arg3];
```

Invocations containing multiple inlined blocks may have their parameter names
left-aligned at a four space indent.

### Function Calls

Use local variables with descriptive names to shorten function calls and reduce
nesting of calls.

```objectivec
// GOOD:

double scoreHeuristic = scores[x] * y + bases[x];
UpdateTally(scoreHeuristic, x, y, z);
```

### Exceptions

Format exceptions with `@catch` and `@finally` labels on the same line as the
preceding `}`. Add a space between the `@` label and the opening brace (`{`), as
well as between the `@catch` and the caught object declaration.

```objectivec
// GOOD:

@try {
  foo();
} @catch (NSException *ex) {
  bar(ex);
} @finally {
  baz();
}
```

### Function Length

Prefer small and focused functions.

Long functions and methods are occasionally appropriate, so no hard limit is
placed on function length. If a function exceeds about 40 lines, think about
whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few
months may add new behavior. This could result in bugs that are hard to find.
Keeping your functions short and simple makes it easier for other people to read
and modify your code.

When updating legacy code, consider also breaking long functions into smaller
and more manageable pieces.

### Vertical Whitespace

Use vertical whitespace sparingly.

To allow more code to be easily viewed on a screen, avoid putting blank lines
just inside the braces of functions.

Limit blank lines to one or two between functions and between logical groups of
code.
