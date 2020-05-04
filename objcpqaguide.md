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

## Principles ([source][principles])

### Optimize for the reader, not the writer ([source][optimize-for-the-reader-not-the-writer])

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

1. Type all arrays, dictionaries, and sets using [Lightweight Generics][generictyping]
2. Remove everything that can be removed from the header file to [Keep the Public API Simple][cleanheaders]
3. [Naming][naming]
   1. _ before ivars.

      ```objc
      BOOL _subsequentAppearance;
      ```

   2. _ before private methods.

      ```objc
      - (void)_processResponse:(PFXResponse *)response;
      ```

   3. No "get" naming on properties or functions.
   4. Limit "with" and "and" in function names.
   5. Include caller as first parameter of all delegate callbacks (see every Apple example).

      ```objc
      // good
      - (void)serverManagerDidFinish:(ATCServerManager *)manager;
      - (void)serverManager:(ATCServerManager *)manager finishedWithError:(NSError *)error;

      // avoid
      - (void)finishedWithError:(NSError *)error;
      ```

   6. Prefix every class.
   7. Prefix every enum name.  Prefix every value with the enum name.
4. Use modern syntax
    - `@[]`, `@{}`, `@()` instead of array/dictionary/number.

      ```objc
      // good
      NSArray *listTitles = @[@"Recent", @"Favorites"]

      // good
      NSDictionary *dataSections = @{
        @4: routeEntrySection,
        @15: errorSection
      }
      ```

5. If your API don't support something, use `NSAssert`.  
    - Crash internally instead of silently returning `nil`.

6. Feel free to make liberal use of class methods, categories, and C methods in **implementation** files to better define the inputs and outputs and remove any unclear side effects a function has.  These tend to also be more unit testable.
    - These are for code organization, not for sharing logic with other classes, so they go in the implementation (.m) file.
7. There are multiple standard patterns for delegating responsibility.  When there is more than one method available that meets your needs, prefer using the one that appears first on this list:
   1. Target/action
   2. Block-based handler
   3. Delegate property
   4. NSNotificationCenter
   5. Key-Value Observing (KVO)

### Restrictions

This is a list of coding restrictions that lead to code that is easier to read and debug.

1. `if` statements should not contain comparisons or parentheses.  Instead store these in local `BOOL` variables with descriptive names.

    ```objc
    BOOL needsProcessing = !data.isIncomplete && data.dirty;
    BOOL shouldProcess = needsProcessing && !serverLoader.isBusy;
    if (shouldProcess) {
      // ...
    }
    ```

2. `return` should always return a local variable.  Function calls or property lookup should first be stored to a local variable.  This makes debugging much easier.  Exceptions include: returning empty arrays, dictionaries, or strings.
  
    ```objc
    // good
    HKRecord *record = [_info.dataElements elementNamed:name];
    return record;

    // avoid
    return [_info.dataElements elementNamed:name];
    ```

3. No web service (`ECF`) calls in the implementation of a `UIViewController`.  Put that into a loader/manager.

    ```objc
    @interface PFXActivityServerManager

    @property (nonatomic) ECFConnection *connection;
    @property (nonatomic) PFXActivityLoadParams *params;
    @property (nonatomic, weak) id<PFXActivityServerManagerDelegate> delegate;

    - (void)beginLoading;
    @end
    ```

4. No just-in-time ivar creation.  Create these in your `init` when possible.

    ```objc
    // good
    - (instancetype)init {
      self = [super init];
      if (self) {
        _infoViewController = [InfoViewController new];
      }
      return self;
    }

    - (void)showInfo {
      [self presentViewController:_infoViewController animated:YES completion:nil];
    }

    // avoid
    - (void)showInfo {
      if (!_infoViewController) {
        // this should be in init
        _infoViewController = [InfoViewController new];
      }
      [self presentViewController:_infoViewController animated:YES completion:nil];
    }
    ```

5. Don't overoptimize.  Optimizations often sacrifice readability and the tradeoff is almost never worth it.

    ```objc
    // good
    - (BOOL)_isOldOS {
      return [[UIDevice currentDevice].systemVersion floatValue] < 10.0;
    }

    // avoid
    - (instancetype)init {
      self = [super init];
      if (self) {
        _isOldOS = [[UIDevice currentDevice].systemVersion floatValue] < 10.0;
      }
      return self;
    }

    // avoid
    NSMutableArray *newArray = [NSMutableArray arrayWithCapacity:oldArray.count];
    ```

6. No more than 7 ivars in `UIViewController` subclasses.

    ```objc
    // avoid
    @implementation PFXConversationTableViewController  {
      NSMutableDictionary *_portraitRowHeights,*_landscapeRowHeights,*_readReceiptsByUser,*_readReceiptsByMessage,*_messageDictionary;
      NSMutableArray *_participants,*_loadMessagesStatusQueue,*_messages,*_groups;
      NSString *_conversationID,*_currentUserKey,*_newMessageToast,*_currentWarningMessage;
      NSDictionary *_typingUserList,*_participantDictionary;
      UIView *_toastMessageView;
      BOOL _allRecordsLoaded,_incrementalLoadTriggered,_isActivityReadOnly,_isRotatingToDifferentOrientation,_isSideMenuHost, _shouldDisplayToastMessage,_currentUserLeft;
      CGFloat _bottomInset;
      UITableViewCell *_loadingCell;
      ChatEventManager *_chatEventManager;
      PFXTypingEventManager *_typingEventManager;
      PFXConversationLoadStatus _status;
      ECFCommand *_ecfCommand;
      PFXDataManager *_sharedDataManager;
      PFXDataManager *_localDataManager;
      HKPatient *_patient;
      HKPatientHeaderSummaryView *_patientView;
      PFXPatient *_chatPatient;
      PFXMessageCell *_heightCalculatingCell;
      PFXSenderMessageCell *_heightCalculatingSenderCell;
      PFXParticipantChangeCell *_heightCalculatingChangeCell;
      PFXSystemMessageCell *_heightCalculatingSystemCell;
      PFXConversationTypingView *_typingUsersCircularPhotoListView;
      PFXInputFooterView *_footerContentView;
      PFXConversationHeaderView *_headerPortraitView;
      HKFixedHeaderView *_fixedHeaderView;
      PFXConversationHeaderView *_errorView;
      UIAlertController *_imageMessageAlertController;
      UIImage *_downArrowIcon;
      UIImage *_patientPhoto;
      PFXConversation *_conversation;
      UCBadgeManager *_badgeManager;
      UIView *_headerLandscapeView;
      NSTimer *_refreshTimer;
      NSTimer *_reloadTimer;
      BOOL _isKeyBoardUp;
      BOOL _leavingConversation;
      BOOL _allowCellSelection;
      UIImage * _lastImageMessage, * _placeholderImageMessage , * _breakTheGlassImage;
      BOOL _patientNeedsBreakTheGlass;
      BOOL _newMessageAdded, _viewAtBottomBeforeCameraPressed;
      BOOL _isOpeningPatient;
      BTGManager *_btgManager;
      UIImagePickerController *_imageMessagePicker;
      HKLoadingShieldView *_disabledView;
      BOOL _isSendExistingImagesEnabled;
      BOOL _isVoipDefined;
      BOOL _linkPhoneNumbers;
      BOOL _conversationHasUnreadMessagesInCanto;
      UDIRProviderPhotoUtils * _providerPhotoUtils;
      NSMutableDictionary * _userIndexRelationDictionary;
      NSMutableSet * _requiredEncounterBTG;
      BOOL _isOpeningMediaViewer;
      __weak PFXConversationContainerViewController *_containerVC;
      UDIRAvailabilityIndicatorSubscriptionManager* _availabilitySubscriber;
      BOOL _deepLinkingAway;
      PFXTappablePhoneNumbersUtility *_tappablePhoneNumbersUtility;

      UIViewController* _addParticipantViewController;
    }
    ```

7. Never override `viewWillLayoutSubviews`, `viewDidLayoutSubviews`, `viewWillUpdateConstraints`, or `viewDidUpdateConstraints`.
    - Putting logic in the UI lifecycle methods of a `UIViewController` makes code difficult to debug and is regularly broken by Apple's SDK and OS updates.  If there is no alternative, prefer using a `UIView` subclass to encapsulate this logic instead of using the `UIViewController` callbacks.
8. Never use **Visual Format Language** for layout constraints.  You should using layout anchors like `safeAreaLayoutGuide`.
    - **Visual Format** language relies on parsing strings at runtime, both of which (parsing strings and relying on runtime behavior) mean errors are more difficult to catch.  **Layout anchors** provides type safety and compile time checking to help ensure mistakes are caught right as they are coded.
9. Never have 4 levels of indentation in a method.  Rarely have 3.
    - It is difficult to read code that has this much nested logic and the most important priciple in software development is to **[Optimize for the reader, not the writer][optimize-for-the-reader-not-the-writer]**
10. Never implement instance level convenience initializers.  You can have class level convenience initializers as long as they only use the public interface of the class.
    - This ensures that it is always valid to call the convenience method `-new` on an object.

    ```objc
    // good
    @interface APPEncounter
    @property (nonatomic) NSString *DAT;
    @end

    @interface APPEncounter (Convenience)
    + (instancetype)initWithDAT:(NSString *)DAT;
    @end

    // avoid
    @interface APPEncounter

    - (instancetype)initWithDAT:(NSString *)DAT;

    @property (nonatomic) NSString *DAT;
    @end
    ```

## Naming cheatsheet

See the full [Naming][naming] section of the [Objective-C Style Guide][objc] for additional details.

### Preferred

```objc
// Avoid non-standard abbreviations
int numberOfErrors = 0;
int completedConnectionsCount = 0;
detailsViewController = [DetailsViewController new];

// Use all capitals for acronyms and initialisms within the name
serviceURL = [NSURL URLWithString:@"https://server/path"];

// Prefix classes, protocols, global constants, and enums with three capital letters
@protocol GTMExampleDelegate <NSObject>
@end

extern NSString *GTMExampleErrorDomain;

// File names reflect the name of the class implementation that they contain
/* GTMExample.h */
@interface GTMExample : NSObject
@end

// File names for categories should include the name of the class being extended
/* UIViewController+GTMCrashReporting.h */
@interface UIViewController (GTMCrashReporting)

// Prefix categories
@property(nonatomic, setter=gtm_setUniqueIdentifier:) int gtm_uniqueIdentifier;
- (nullable NSData *)gtm_encodedState;
@end

// Method name should read like a sentence
- (void)addTarget:(id)target action:(SEL)action; // no conjunction needed
// Use "with", "from", and "to" in the second and later parameter names only where necessary
- (CGPoint)convertPoint:(CGPoint)point fromView:(UIView *)view; // conjunction clarifies parameter
- (void)replaceCharactersInRange:(NSRange)aRange
            withAttributedString:(NSAttributedString *)attributedString;

// Methods that returns an object should beginning with a noun identifying the object
- (Sandwich *)sandwich;

// Accessors should be named the same as the object it's getting
- (id)delegate;

// It should not be prefixed with the word `get`
// - (id)getDelegate;  // AVOID

// Local variables use camel case
int myLocalVariable;

// Instance variables use camel case and begin with an underscore
UITextField *_usernameTextField;

// Private functions begin with an underscore
- (void)_performInternalAction;

// Prefix constants like `ClassNameConstantName` or `ClassNameEnumName`
typedef NS_ENUM(NSInteger, WRDDisplayTinge) {
  // For swift interop, enumerated values should have names that extend the typedef name
  WRDDisplayTingeGreen = 1,
  WRDDisplayTingeBlue = 2,
};

// Use k for constants of static storage declared within implementation files
static const int kFileCount = 12;
static NSString *const kUserKey = @"kUserKey";
```

## Cocoa and Objective-C Features

### Identify Designated Initializer ([source][identify_designated_init])

Clearly identify your designated initializer using the `NS_DESIGNATED_INITIALIZER` macro.

### Override Designated Initializer ([source][override_designated_init])

When writing a subclass that requires an `init...` method, make sure you
override the designated initializer of the superclass. If you fail to override
the designated initializer of the superclass, your initializer may not be called
in all cases, leading to subtle and very difficult to find bugs.

### Keep the Public API Simple ([source][cleanheaders])

Keep your class simple; avoid "kitchen-sink" APIs. If a method doesn't need to
be public, keep it out of the public interface.

### Setters copy NSStrings ([source][setters_copy_nsstrings])

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

### Use Lightweight Generics to Document Contained Types ([source][generictyping])

Every `NSArray`, `NSDictionary`, or `NSSet` reference should be declared using
lightweight generics for improved type safety and to explicitly document usage.

```objectivec
// GOOD:

@property(nonatomic, copy) NSArray<Location *> *locations;
@property(nonatomic, copy, readonly) NSSet<NSString *> *identifiers;

NSMutableArray<MyLocation *> *mutableLocations = [otherObject.locations mutableCopy];
```

### Nullability ([source][nullability])

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

### Delegate Pattern ([source][delegate_pattern])

Delegates, target objects, and block pointers should not be retained when doing
so would create a retain cycle.  Instead, prefer to weakly capture these references.

## Spacing and Formatting

### Spaces vs. Tabs ([source][spaces_vs_tabs])

Use only spaces, and indent 4 spaces at a time. We use spaces for indentation.
Do not use tabs in your code.

You should set your editor to emit spaces when you hit the tab key, and to trim
trailing spaces on lines.

### Line Length ([source][line_length])

The maximum line length for Objective-C files is 100 columns.

You can make violations easier to spot by enabling *Preferences > Text Editing >
Page guide at column: 100* in Xcode.

### Method Declarations and Definitions ([source][method_declarations_and_definitions])

One space should be used between the `-` or `+` and the return type, and no
spacing in the parameter list except between parameters and any * in the parameter typing.

Methods should look like this:

```objectivec
// GOOD:

- (void)doSomethingWithString:(NSString *)theString {
  ...
}
```

### Function Length ([source][function_length])

Prefer small and focused functions.

Long functions and methods are occasionally appropriate, so no hard limit is
placed on function length.  If a function exceeds about 40 lines, think about
whether it can be broken up without harming the structure of the program.

Even if your long function works perfectly now, someone modifying it in a few
months may add new behavior. This could result in bugs that are hard to find.
Keeping your functions short and simple makes it easier for other people to read
and modify your code.

When updating legacy code, consider also breaking long functions into smaller
and more manageable pieces.

[objc]: objcguide.md
[principles]: objcguide.md#principles
[optimize-for-the-reader-not-the-writer]: objcguide.md#optimize-for-the-reader-not-the-writer
[generictyping]: objcguide.md#use-lightweight-generics-to-document-contained-types
[cleanheaders]: objcguide.md#keep-the-public-api-simple
[naming]: objcguide.md#naming
[identify_designated_init]: objcguide.md#identify-designated-initializer
[override_designated_init]: objcguide.md#override-designated-initializer
[setters_copy_nsstrings]: objcguide.md#setters-copy-nsstrings
[nullability]: objcguide.md#nullability
[delegate_pattern]: objcguide.md#delegate-pattern
[spaces_vs_tabs]: objcguide.md#spaces-vs-tabs
[line_length]: objcguide.md#line-length
[method_declarations_and_definitions]: objcguide.md#method-declarations-and-definitions
[function_length]: objcguide.md#function-length
