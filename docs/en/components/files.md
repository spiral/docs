# Work with Files
`FilesInterface` represents set of common filesystem operation (use components like [Flysystem](https://github.com/thephpleague/flysystem) if you wish to have full filesystem abstraction). 

> Check [Storage component](/storage/overview.md) if you want to work with your files regarding their location.

## FilesInterface
```php
interface FilesInterface
{
    //Owner and group can write
    const RUNTIME = 0665;

    //Only owner can write
    const READONLY = 0655;

    /**
     * Few size constants for better size manipulations.
     */
    const KB = 1024;
    const MB = 1048576;
    const GB = 1073741824;

    /**
     * Default location (directory) separator.
     */
    const SEPARATOR = '/';

    /**
     * Ensure location (directory) existence with specified mode.
     *
     * @param string $directory
     * @param int    $mode When NULL class can pick default mode.
     *
     * @return bool
     */
    public function ensureDirectory(string $directory, int $mode = null): bool;

    /**
     * Read file content into string.
     *
     * @param string $filename
     *
     * @return string
     *
     * @throws FileNotFoundException
     */
    public function read(string $filename): string;

    /**
     * Write file data with specified mode. Ensure location option should be used only if desired
     * location may not exist to ensure such location/directory (slow operation).
     *
     * @param string $filename
     * @param string $data
     * @param int    $mode            When NULL class can pick default mode.
     * @param bool   $ensureDirectory Ensure final destination!
     *
     * @return bool
     *
     * @throws WriteErrorException
     */
    public function write(
        string $filename,
        string $data,
        int $mode = null,
        bool $ensureDirectory = false
    ): bool;

    /**
     * Same as write method with will append data at the end of existed file without replacing it.
     *
     * @see write()
     *
     * @param string $filename
     * @param string $data
     * @param int    $mode When NULL class can pick default mode.
     * @param bool   $ensureDirectory
     *
     * @return bool
     *
     * @throws WriteErrorException
     */
    public function append(
        string $filename,
        string $data,
        int $mode = null,
        bool $ensureDirectory = false
    ): bool;

    /**
     * Method has to return local uri which can be used in require and include statements.
     * Implementation is allowed to use virtual stream uris if it's not local.
     *
     * @param string $filename
     *
     * @return string
     */
    public function localFilename(string $filename): string;

    /**
     * Delete local file if possible. No error should be raised if file does not exists.
     *
     * @param string $filename
     */
    public function delete(string $filename);

    /**
     * Delete directory all content in it.
     *
     * @param string $directory
     * @param bool   $contentOnly
     */
    public function deleteDirectory(string $directory, bool $contentOnly = false);

    /**
     * Move file from one location to another. Location must exist.
     *
     * @param string $filename
     * @param string $destination
     *
     * @return bool
     *
     * @throws FileNotFoundException
     */
    public function move(string $filename, string $destination): bool;

    /**
     * Copy file at new location. Location must exist.
     *
     * @param string $filename
     * @param string $destination
     *
     * @return bool
     *
     * @throws FileNotFoundException
     */
    public function copy(string $filename, string $destination): bool;

    /**
     * Touch file to update it's timeUpdated value or create new file. Location must exist.
     *
     * @param string $filename
     * @param int    $mode When NULL class can pick default mode.
     */
    public function touch(string $filename, int $mode = null);

    /**
     * Check if file exists.
     *
     * @param string $filename
     *
     * @return bool
     */
    public function exists(string $filename): bool;

    /**
     * Get filesize in bytes if file does exists.
     *
     * @param string $filename
     *
     * @return int
     *
     * @throws FileNotFoundException
     */
    public function size(string $filename): int;

    /**
     * Get file extension using it's name. Simple but pretty common method.
     *
     * @param string $filename
     *
     * @return string
     */
    public function extension(string $filename): string;

    /**
     * Get file MD5 hash.
     *
     * @param string $filename
     *
     * @return string
     *
     * @throws FileNotFoundException
     */
    public function md5(string $filename): string;

    /**
     * Timestamp when file being updated/created.
     *
     * @param string $filename
     *
     * @return int
     *
     * @throws FileNotFoundException
     */
    public function time(string $filename): int;

    /**
     * @param string $filename
     *
     * @return bool
     */
    public function isDirectory(string $filename): bool;

    /**
     * @param string $filename
     *
     * @return bool
     */
    public function isFile(string $filename): bool;

    /**
     * Current file permissions (if exists).
     *
     * @param string $filename
     *
     * @return int|bool
     *
     * @throws FileNotFoundException
     */
    public function getPermissions(string $filename): int;

    /**
     * Update file permissions.
     *
     * @param string $filename
     * @param int    $mode
     *
     * @throws FileNotFoundException
     */
    public function setPermissions(string $filename, int $mode);

    /**
     * Flat list of every file in every sub location. Locations must be normalized.
     *
     * Note: not a generator yet, waiting for PHP7.
     *
     * @param string $location Location for search.
     * @param string $pattern  Extension pattern.
     *
     * @return array
     */
    public function getFiles(string $location, string $pattern = null): array;

    /**
     * Return unique name of temporary (should be removed when interface implementation destructed)
     * file in desired location.
     *
     * @param string $extension Desired file extension.
     * @param string $location
     *
     * @return string
     */
    public function tempFilename(string $extension = '', string $location = null): string;

    /*
     * Move outside in a future versions.
     */

    /**
     * Create the most normalized version for path to file or location.
     *
     * @param string $path        File or location path.
     * @param bool   $asDirectory Path points to directory.
     *
     * @return string
     */
    public function normalizePath(string $path, bool $asDirectory = false): string;

    /**
     * Get relative location based on absolute path.
     *
     * @param string $path Original file or directory location (to).
     * @param string $from Path will be converted to be relative to this directory (from).
     *
     * @return string
     */
    public function relativePath(string $path, string $from): string;
}
```

Component is available using shortcut "files":
```php
protected function indexAction()
{
    dump($this->files->getFiles(directory('runtime'), 'php'));
}
```

> Component is based on `Symfony/Finder`, you can use in your code directly as well.
