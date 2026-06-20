# Symfony: The Fast Track

This repository holds the content of the book [Symfony: The Fast
Track](https://symfony.com/book), the official guide to building a Symfony
application step by step.

It contains the book text only. The tooling that turns this content into HTML
and PDF, translates it, and runs its code as a test suite lives in a separate
repository.

## How the content is organized

Files are stored as `<version>/<locale>/*.rst`, for example
`8.1/en/7-database.rst`:

* `<version>` is the Symfony version the edition targets (e.g. `8.1`).
* `<locale>` is the language code (e.g. `en`, `fr`, `de`).
* English (`en`) is the source language: every other locale is a translation
  of it.

Code, directives, references, and file paths are identical across all locales of
a given version; only the prose and the heading underlines differ.

## Contributing

We welcome contributions, but please read the rules below before opening a pull
request. They keep the book consistent and reviewable across all languages.

### Improving translations

* Tweaking and polishing existing translations is exactly what we are looking
  for.
* Only contribute to a language if you are a native speaker.
* A translation change is merged only after a few approvals from native
  speakers.

### Improving the English text

* Improving the English source is welcome too, just like any other language.

### Fixing bugs

* Fixing bugs in the content or in the code is welcome.

### New translations

* No new translation language is allowed by default; we only maintain the
  existing ones.
* Exceptions are possible but must be validated by the core team in a ticket
  after discussion.

### Content changes

* Changing the content or adding new chapters is not allowed.
* If you have an idea, open an issue to discuss it.

### LLM contributions

* LLM contributions are not allowed by default, in particular automated
  translations.

## License

The book content is copyrighted. See the book's website for details.
