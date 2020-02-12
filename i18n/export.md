# Internalization - Import and Export
It is possible to export application locale bundles into following formats:
- raw PHP
- GetText PO
- CSV
- JSON

## Export Locale
To export application locate bundles run:

```bash
$ php app.php i18n:export en ./
```

> Export `en` locale into current directory.

You should observe file `messages.en.php` created in this directory. To export in alternative formats:

```bash
$ php app.php i18n:export en ./ -d po
```

This command will export the locale into gettext format.

## Generate Locale
The framework is capable to automatically generate locale files using static code indexation. Run command `i18n:index` 
to find all declared stings.

```bash
$ php app.php i18n:index -vv
```

## Import Locale
To import locate into the project simply locate the locale files in `app/locale/{lang}` directory.