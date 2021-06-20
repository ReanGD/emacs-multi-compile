# multi-compile
Multi target interface to compile.

## Screenshot

![multi-target](https://cloud.githubusercontent.com/assets/1151286/10209424/de607546-67e3-11e5-8cb0-f50d390e823b.png)

## Installation

You can install `multi-compile.el` from [MELPA](https://melpa.org/) with `package.el`

```
 M-x package-install RET multi-compile RET
```

Or drop `multi-compile.el` and [`dash.el`](https://github.com/magnars/dash.el) into your load path.

## Common settings format:
```lisp
(setq multi-compile-alist '(
    (Trigger1 . (("menu item 1" . "command 1")
                ("menu item 2" . "command 2")
                ("menu item 3" . "command 3")))

    (Trigger2 . (("menu item 4" "command 4" (expression returns a directory for the compilation))
                 ("menu item 5" "command 5" (expression returns a directory for the compilation))))
...
    ))
```

In addition to the common string command, there is also an extended syntax that can be used for long or complicated commands.
If the command consists of a list, the elements of the list are expected to be strings and will be joined with a space.
This is a convenience syntax that may be desirable to avoid extremely long command strings.
For instance, if the command consists of long paths or many options, it may be desirable that each path or option exist on its own line.
This extended command syntax is supported both with and without the optional compilation directory expression.

```lisp
(setq multi-compile-alist '(
    (Trigger1 . (("menu item 1" . "command 1")
                 ("menu item 2" ("application 2"
                                 "option 1"
                                 "option 2"
                                 ...))
                 ("menu item 3" . "command 3")))

    (Trigger2 . (("menu item 4" "command 4" (expression returns a directory for the compilation))
                 ("menu item 5" ("application 5"
                                 "option 1"
                                 "option 2"
                                 ...)
                                (expression returns a directory for the compilation))))
...
    ))
```

### Triggers:

- Can be major-mode:

Example adds 3 a menu item for rust-mode:
```lisp
(setq multi-compile-alist '(
    (rust-mode . (("rust-debug" . "cargo run")
                  ("rust-release" . "cargo run --release")
                  ("rust-test" . "cargo test")))
    ...
    ))
```

- Can be file/buffer name pattern:

Adds menu item "print-filename" for filename ends "txt":
```lisp
(setq multi-compile-alist '(
    ("\\.txt\\'" . (("print-filename" . "echo %file-name")))
    ...
    ))
```
menu item "print-hello" for scratch buffer:
```lisp
(setq multi-compile-alist '(
    ("*scratch*" . (("print-hello" . "echo 'hello'")))
    ...
    ))
```
adds "item-for-all" for any file or buffer:
```lisp
(setq multi-compile-alist '(
    ("\\.*" . (("item-for-all" . "echo 'I am item for all'")))
    ...
    ))
```

- Can by expression returns t or nil:

adds "item-for-home" for any file in "/home/" directory:
```lisp
(defun string/starts-with (string prefix)
    "Return t if STRING starts with prefix."
    (and (stringp string) (string-match (rx-to-string `(: bos ,prefix) t) string)))

(setq multi-compile-alist '(
    ((string/starts-with buffer-file-name "/home/") . (("item-for-home" . "echo %file-name")))
    ...
    ))
```
### Commands:
String with command for "compile" package.
In a compilation commands, you can use the templates:

- "%path" - "/tmp/prj/file.rs" (full path)
- "%dir" - "/tmp/prj/"
- "%file-name" - "file.rs"
- "%file-sans" - "file"
- "%file-ext" - "rs"
- "%make-dir" - (Look up the directory hierarchy from current file for a directory containing "Makefile") - "/tmp/"

### Advanced Commands:
In their basic form, template variables correspond to expressions that are evaluated and their results substituted into the compile command.
These basic template variables exist in the expansion list contained in `multi-compile-template`, but can be customized as desired.
This means they can be replaced or additional template variables added to the default set.
In addition to expressions, template variables can also correspond to prompts and functions.
To distinguish between these different types of template variables, the following terms will be used when distinction is important:
  - Expression template variable
  - Prompt template variable
  - Function template variable

There are two different types of prompts supported, a string prompt and a choice prompt.
Prompts utilize a descriptive prompt string to display to the user as well as allowing for either free-form input
(i.e., string prompt) or selecting from a list of possible choices (i.e., choice prompt).
Prompt template variables may appear in the command string multiple times, but the user will only be prompted once, and each
instance of the prompt template variable will be replaced with the user input.

In addition, function template variables are also supported which can be used to perform arbitrary transformation of the text.
They can take zero or more arguments and can further manipulate the text of the compile command.
It may not be obvious, but template variable strings are actually regular expressions.
In their basic form, no regular expression syntax is needed and the literal characters of the variable are matched.
However, when used to represent functions, sub-expressions of matches of the function template variable are interpreted as the parameters for the function.
Furthermore, function template variables can be nested so that they can be stacked and applied as desired.
These represent advanced features of the template variable structure which can support building of complex compilation commands when necessary.

The following are examples which demonstrate the advanced prompt template variables and function template variables capability.
These can easily be experimented with to gain further understanding on how to apply them for your own needs.
These examples are structured so that they can be placed into the \*scratch\* buffer and evaluated and then `M-x multi-compile-run` can be executed without affecting your global settings.
Once you become familiar with the syntax and power of these advanced capabilities, you can easily incorporate them into your own configuration.

The more advanced and complicated the interaction of the function template variables become, the easier it is to start having problems due to order of expansion.
These examples use '[' and ']' to surround the function parameters, but this is only shown for convention and a different approach could be used.
Furthermore, the sub-expressions which are defined for the function parameters don't allow a '[' to exist in the sub-expression and also use the non-greedy (i.e., '?') operator.
This is to make sure that inner nested functions are expanded prior to outer functions (by preventing a match when unexpanded functions exist).
Function expansion is attempted after all expression template variables and prompt template variables have been expanded.
The reason for this is that expressions and prompts can be expanded in a single pass, however functions may require multiple passes to fully expand, due to potential nesting.

To help in making sure the function template variables receive the type of parameters that you expect, it is suggested to perform checks on the input parameters.
This is not required, but can be helpful to isolate problems of expansion order when initially incorporating this functionality.
The following examples use `cl-assert` to perform the input validation to make sure the functions are receiving the data that was expected.

The way in which the type of a template variable is determined is based on the content in `multi-compile-template`.
The content of the `multi-compile-template` corresponds to an association list.
The template variable string representation corresponds to the key of the association list.
The associated value of the association list is used to determine the type of the template variable.
  - Prompt template variable
    - If the associated value is a string, the variable is interpreted as a string prompt variant of the prompt template variable.
      The string is used as the prompt displayed to the user for entering of the free-form text.
    - If the associated value is a cons cell with the `car` of the cons cell being a string, then the variable is interpreted as a choice prompt variant of the prompt template variable.
      The string is used as the prompt displayed to the user for choosing from the set of values.
      The `cdr` of the cons cell is used to represent the list of possible values to choose from.
      The configured completion system (as determined by `multi-compile-completion-system`) will be used to prompt the user to make a selection.
  - Function template variable
    - If the associated value is a function, the variable is interpreted as a function template variable.
      It is expected that the supplied function parameters align with the sub-expressions found in the template variable string.
  - Expression template variable
    - If the associated value does not match any of the previous types, it is evaluated as an expression.

```lisp
(setq-local multi-compile-alist
            '(("*scratch*" . (("Favorite Color" . "echo 'I like %favorite-color too, in fact %uc[%favorite-color] is also my favorite!'")
                              ("Choose Color" . "echo 'That's too bad, I don't like %uc[%choose-color], I like %uc[%other-color[%other-color[%choose-color]]] instead!'")
                              ("Add Numbers" . "echo '%enter-first-number + %enter-second-number = %add[%enter-first-number,%enter-second-number]'")
                              ))))

(setq-local multi-compile-template
            '(;; string prompt example to enter free-form text
              ("%favorite-color" . "What's your favorite color?: ")
              ;; choice prompt example to select from a set of options
              ("%choose-color" . ("Choose one of these colors: " . ("Red" "Green" "Blue")))
              ;; function example to uppercase the matched sub-expression
              ("%uc\\[\\([^\\[]+?\\)]" . upcase)
              ;; function example to perform a computation with the matched sub-expression
              ("%other-color\\[\\([^\\[]+?\\)]" . (lambda (color)
                                                    (cl-assert (member color '("Red" "Green" "Blue")))
                                                    (cond ((string= color "Red") "Green")
                                                          ((string= color "Green") "Blue")
                                                          ((string= color "Blue") "Red"))))
              ;; function example utilizing two parameters
              ("%enter-first-number" . "Enter first number: ")
              ("%enter-second-number" . "Enter second number: ")
              ("%add\\[\\(.+?\\),\\(.+?\\)]". (lambda (first second)
                                                (cl-assert (string-match "^[0-9]+\\(\\.[0-9]+\\)?\\'" first))
                                                (cl-assert (string-match "^[0-9]+\\(\\.[0-9]+\\)?\\'" second))
                                                (number-to-string (+ (string-to-number first)
                                                                     (string-to-number second)))))))
```

### Compilation directory:
Default compilation directory is a system variable default-directory.
If you want to change it, add an expression that returns the correct directory.

For example, add a menu to compile go project under git:
```lisp
(setq multi-compile-alist '(
    (go-mode . (("go-build" "go build -v"
                 (locate-dominating-file buffer-file-name ".git"))
                ("go-build-and-run" "go build -v && echo 'build finish' && eval ./${PWD##*/}"
                 (multi-compile-locate-file-dir ".git"))))
    ...
    ))
```
Where:

  - "${PWD##*/}" returns current directory name.
  - (locate-dominating-file buffer-file-name ".git") - look up the directory hierarchy from current file for a directory containing directory ".git"
  - (multi-compile-locate-file-dir ".git") this is equivalent to (locate-dominating-file buffer-file-name ".git")

## Usage

- M-x multi-compile-run

## Links

[In Russian](http://reangdblog.blogspot.com/2015/10/emacs-multi-compile.html)

[In Russian (Part 2)](http://reangdblog.blogspot.com/2016/02/emacs-multi-compile.html)
