# Internalization - Import and Export
It is possible to export application locale bundles into the following formats:
- raw PHP
- GetText PO
- CSV
- JSON

## Export Locale
To export application locate bundles run:

```bash
$ php app.php i18n:export en ./
```

> Export `en` locale into the current directory.

You should observe file `messages.en.php` created in this directory. To export in alternative formats:

```bash
$ php app.php i18n:export en ./ -d po
```

This command will export the locale into the GetText format.

## Generate Locale
The framework is capable of generating locale files using static code indexation automatically. Run command `i18n:index` 
to find all declared stings.

```bash
$ php app.php i18n:index -vv
```

## Import Locale
Import locate into the project by placing files `app/locale/{lang}` directory. Use GetText, PHP, or JSON formats.
