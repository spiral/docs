# Work with local Files
Spiral Files component and `FilesInterface` are intended to simplify work with local harddrive rather than create a universal filesystem abstraction (like [Flysystem](https://github.com/thephpleague/flysystem)). 

If you want to use abstract storage and work with remote locations (like Amazon, Rackspace, FTP, SFTP, GridFS) you might need to look at spiral [Storage Manager component](/storage/overview.md) which is responsible for such abstractions or install already mentioned Flysystem. 

## FilesInterface
Like any other component, the default implementation of spiral FileManager is based on the interface. Let's view the interface source code to better understand what methods are available:

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
     * @param string $directory
     * @param int    $mode
     * @return bool
     */
    public function ensureDirectory($directory, $mode = self::RUNTIME);

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
     * location may not exist to ensure such location/directory (slow operation).
     *
     * @param string $filename
     * @param string $data
     * @param int    $mode            One of mode constants.
     * @param bool   $ensureDirectory Ensure final destination!
     * @return bool
     * @throws WriteErrorException
     */
    public function write($filename, $data, $mode = null, $ensureDirectory = false);

    /**
     * Same as write method with will append data at the end of existed file without replacing it.
     *
     * @see write()
     * @param string $filename
     * @param string $data
     * @param int    $mode
     * @param bool   $ensureDirectory
     * @return bool
     * @throws WriteErrorException
     */
    public function append($filename, $data, $mode = null, $ensureDirectory = false);

    /**
     * Method has to return local uri which can be used in require and include statements.
     * Implementation is allowed to use virtual stream uris if it's not local.
     *
     * @param string $filename
     * @return string
     */
    public function localUri($filename);

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
     * @param string $filename
     * @return bool
     */
    public function isDirectory($filename);

    /**
     * @param string $filename
     * @return bool
     */
    public function isFile($filename);

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
     * Note: not a generator yet, waiting for PHP7.
     *
     * @param string $location Location for search.
     * @param string $pattern  Extension pattern.
     * @return array
     */
    public function getFiles($location, $pattern = null);

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
     * @param string $path      File or location path.
     * @param bool   $directory Path points to directory.
     * @return string
     */
    public function normalizePath($path, $directory = false);

    /**
     * Get relative location based on absolute path.
     *
     * @param string $path Original file or directory location (to).
     * @param string $from Path will be converted to be relative to this directory (from).
     * @return string
     */
    public function relativePath($path, $from);
}
```

> Inteface is missing deleteDirectory method which is going to be added soon.

The only notable part of FileInterface and it's default implementation - FileManager's ability to ensure the file location (directory) with the requested file mode. This functionality is used all across the spiral core to simplify file writing operations and be less dependent on a specific directory structure. Let's look at some examples (and again, we can either use FilesInterface as our dependency or the short binding "files"):

```php
protected function indexAction(FilesInterface $files)
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

Another pretty common operation is the `getFiles` method which is used to recursively read the content of the specified directory:

```php
protected function indexAction()
{
    dump($this->files->getFiles(directory('runtime'), 'php'));
}
```

> Files are based on Symfony/Finder components which you can use in your code directly as well.
