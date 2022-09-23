# markdown-site

## [Linux](https://ubuntu.com/)
## [Google](https://google.com/)
## [Microsoft](https://microsoft.com/)

https://github.com/gurug-prog/markdown-site/blob/c7a0d5985e0bfc694a3b0368dd1bb138c64561a5/README.md?plain=1#L1-L5

```cs
using System.Collections;
using System.IO.Abstractions;
using Newtonsoft.Json;

namespace FileStorage;

public class FileList<T> : IEnumerable<T>
{
    private readonly IFileSystem _fileSystem;
    private string _filepath;

    private string FilePath
    {
        get
        {
            EnsureFileExists();
            return _filepath;
        }
    }

    public FileList(string filepath, IFileSystem fileSystem)
    {
        _filepath = filepath;
        _fileSystem = fileSystem;
    }

    private void EnsureFileExists()
    {
        if (_fileSystem.File.Exists(_filepath)) return;
        string directory = _fileSystem.Path.GetDirectoryName(_filepath);
        _fileSystem.Directory.CreateDirectory(directory);
        _fileSystem.File.Create(_filepath).Dispose();
    }

    private static T? DeserializeLine(string? line)
        => JsonConvert.DeserializeObject<T>(line);

    private static string SerializeObject(T obj)
        => JsonConvert.SerializeObject(obj);

    private IEnumerable<T> EnumerateEntries()
    {
        foreach (var line in _fileSystem.File.ReadLines(FilePath))
        {
            var obj = DeserializeLine(line);
            if (obj == null) continue;
            yield return obj;
        }
    }

    public IEnumerator<T> GetEnumerator()
    {
        return EnumerateEntries().GetEnumerator();
    }

    IEnumerator IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }

    public void Add(T objectToAdd)
    {
        string json = SerializeObject(objectToAdd);
        string line = $"{json}{Environment.NewLine}";
        _fileSystem.File.AppendAllText(FilePath, line);
    }

    private IEnumerable<T> Remove(Func<T, bool> predicate, int? limit)
    {
        var removed = new List<T>();
        string? tempFile = null;
        string? line = null;
        string filePath = FilePath;
        try
        {
            tempFile = _fileSystem.Path.GetTempFileName();
            using (var sr = new StreamReader(_fileSystem.FileStream.Create(filePath, FileMode.Open)))
            using (var sw = new StreamWriter(_fileSystem.FileStream.Create(tempFile, FileMode.Create)))
            {
                int deleted = 0;
                while ((line = sr.ReadLine()) != null)
                {
                    var obj = DeserializeLine(line);
                    if (obj == null) continue;
                    if ((limit == null || deleted < limit) && predicate(obj))
                    {
                        removed.Add(obj);
                        deleted++;
                    }
                    else
                        sw.WriteLine(line);
                }
            }
            

            _fileSystem.File.Delete(filePath);
            _fileSystem.File.Move(tempFile, filePath);
        }
        finally
        {
            if (tempFile != null && _fileSystem.File.Exists(tempFile))
                _fileSystem.File.Delete(tempFile);
        }

        return removed;
    }

    public void RemoveAll(Func<T, bool> predicate)
        => Remove(predicate, null);

    public void RemoveFirst(Func<T, bool> predicate)
        => Remove(predicate, 1);

    public void EditFirst(Func<T, bool> predicate, Func<T, T> map)
    {
        var removed = Remove(predicate, 1)
            .FirstOrDefault();
        if (removed == null) return;
        var edited = map(removed);
        Add(edited);
    }
}
```
