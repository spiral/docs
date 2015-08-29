# Work with local Files
Spiral Files component and `FilesInterface` are intendend to simplify work with local harddriver rather than create universal filesystem abstraction (like [Flysystem](https://github.com/thephpleague/flysystem)). If you want to use with abstract storage and work with remote locations (like Amazon, Rackspace, FTP, SFTP, GridFS) you might need to look at spiral [Storage Manager component](/storage/overview.md) which is responsible for such abstractions. 

## FilesInterface
As any other component default imlementation of spiral FileManager is based on interface. Let's view such interface source to better understand what methods are available for us:

```php
interface FilesInterface
{
    /**
     * Permission mode: fully writable and readable files.
     */
    const RUNTIME = 0777;

    /**
     * Permission mode: only locked to parent (one environment).
     */
    const READONLY = 0666;

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
     * @param string $location
     * @param int    $mode
     * @return bool
     */
    public function ensureLocation($location, $mode = self::RUNTIME);

    /**
     * Read file content into string.
     *
     * @param string $filename
     * @return string
     * @throws FileNotFoundException
     */
    public function read($filename);

    /**
     * Write file data with specified mode. Ensure location option should be used only if desired
     * location may not exist to ensure sush location/directory (slow operation).
     *
     * @param string $filename
     * @param string $data
     * @param int    $mode           One of mode constants.
     * @param bool   $ensureLocation Ensure final destination!
     * @return bool
     * @throws WriteErrorException
     */
    public function write($filename, $data, $mode = null, $ensureLocation = false);

    /**
     * Same as write method with will append data at the end of existed file without replacing it.
     *
     * @see write()
     * @param string $filename
     * @param string $data
     * @param int    $mode
     * @param bool   $ensureLocation
     * @return bool
     * @throws WriteErrorException
     */
    public function append($filename, $data, $mode = null, $ensureLocation = false);

    /**
     * Delete local file if possible. No error should be raised if file does not exists.
     *
     * @param string $filename
     */
    public function delete($filename);

    /**
     * Move file from one location to another. Location must exist.
     *
     * @param string $filename
     * @param string $destination
     * @return bool
     * @throws FileNotFoundException
     */
    public function move($filename, $destination);

    /**
     * Copy file at new location. Location must exist.
     *
     * @param string $filename
     * @param string $destination
     * @return bool
     * @throws FileNotFoundException
     */
    public function copy($filename, $destination);

    /**
     * Touch file to update it's timeUpdated value or create new file. Location must exist.
     *
     * @param string $filename
     * @param int    $mode
     */
    public function touch($filename, $mode = null);

    /**
     * Check if file exists.
     *
     * @param string $filename
     * @return bool
     */
    public function exists($filename);

    /**
     * Get filesize in bytes if file does exists.
     *
     * @param string $filename
     * @return int
     * @throws FileNotFoundException
     */
    public function size($filename);

    /**
     * Get file extension using it's name. Simple but pretty common method.
     *
     * @param string $filename
     * @return string
     */
    public function extension($filename);

    /**
     * Get file MD5 hash.
     *
     * @param string $filename
     * @return string
     * @throws FileNotFoundException
     */
    public function md5($filename);

    /**
     * Timestamp when file being updated/created.
     *
     * @param string $filename
     * @return int
     * @throws FileNotFoundException
     */
    public function time($filename);

    /**
     * Current file permissions (if exists).
     *
     * @param string $filename
     * @return int|bool
     * @throws FileNotFoundException
     */
    public function getPermissions($filename);

    /**
     * Update file permissions.
     *
     * @param string $filename
     * @param int    $mode
     * @throws FileNotFoundException
     */
    public function setPermissions($filename, $mode);

    /**
     * Flat list of every file in every sub location. Locations must be normalized.
     *
     * @param string            $location  Location for search.
     * @param null|string|array $extension Extension or array of extensions to files.
     * @return array
     */
    public function getFiles($location, $extension = null);

    /**
     * Return unique name of temporary (should be removed when interface implementation destructed)
     * file in desired location.
     *
     * @param string $extension Desired file extension.
     * @param string $location
     * @return string
     */
    public function tempFilename($extension = '', $location = null);

    /**
     * Create the most normalized version for path to file or location.
     *
     * @param string $path File or location path.
     * @return string
     */
    public function normalizePath($path);

    /**
     * Get relative location based on absolute path.
     *
     * @param string $path      Original file or directory location (to).
     * @param string $directory Path will be converted to be relative to this directory (from).
     * @return string
     */
    public function relativePath($path, $directory);
}
```

The only notable part of FileInterface and default implementation - FileManager is ability to ensure file location (directory) with requested file mode. This functionality used all across spiral core to simplify file writing operations and be less depenendend on specific directory structure. Let's view some example (and again, we can either use FilesInterface as our dependency or short binding "files"):

```php
public function index(FilesInterface $files)
{
    //Directory runtime/folder will be automatically created with RUNTIME (777) mode
    $files->write(
        directory('runtime') . '/folder/file.txt',
        'data',
        FilesInterface::RUNTIME,
        true
    );

    dump($this->files->read(directory('runtime') . '/folder/file.txt'));
}
```