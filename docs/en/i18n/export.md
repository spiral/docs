# Internalization - Import and Export

It is possible to export application locale bundles into the following formats:
- PHP
- GetText PO
- CSV
- JSON

## Export Locale

To export application locate bundles run:

```bash
php app.php i18n:export en ./
```

> **Note**
> Export the locale `en`  into the current directory.

You should see the file `messages.en.php` created in this directory. To export in alternative formats:

```bash
php app.php i18n:export en ./ -d po
```

This command will export the locale into the `GetText` format.

## Generate Locale

The framework is able to automatically generate locale files using static code indexation. Run the command `i18n:index` to find all the declared stings.

```bash
php app.php i18n:index -vv
```

## Import Locale

Import thelocate into the project by placing the files `app/locale/{lang}` directory. Use the `GetText`, `PHP`, or `JSON` formats.
