[![Clojars](http://clojars.org/iota/latest-version.svg)](http://clojars.org/iota)
[![Build Status](https://api.travis-ci.org/thebusby/iota.png?branch=master)](https://travis-ci.org/thebusby/iota)


# Iota

Iota is a Clojure library for handling large text files in memory, and offers the following benefits;
* Tuned for Clojure's reducers, letting you reduce over large files quickly.
* Uses Java NIO's mmap() for rapid IO and handling files larger than available memory.
* Efficiently stores data as it is represented in the file, and only converts to java.lang.String when necessary.
* Offers efficient indexing and caching that emulates Clojure's native vector and seq data structures.
* Adjustable buffer sizes for IO and caching, enabling tuning for specific data sets.
 
 
Why write this library?  
I wanted to be able to use Clojure reducers against large text files to speed up data processing, and without needing more than 10% memory overhead. Due to Java's inefficient storage of [Strings](http://www.javamex.com/tutorials/memory/string_memory_usage.shtml), I found that a 1GB TSV file consumed 10GB of RAM when loaded line by line into a Clojure vector. 


## Details

Iota offers `iota/seq` and `iota/vec` for two different use cases.
Both treat a line as delimited by a byte separator (default is newline, value 10 decimal), as an element.

Note that this is slightly different than Clojure's
[`line-seq`](https://clojuredocs.org/clojure.core/line-seq).
`line-seq` uses Java's
[`readLine()`](https://docs.oracle.com/javase/8/docs/api/java/io/BufferedReader.html#readLine--)
method, which is hard-coded to treat any of a newline, or a carriage
return, or a carriage return followed immediately by a newline, as a
line separator.

Differences | iota/seq | iota/vec
--- | --- | ---
On creation | Quick, mmap's the file, and stops | Slow, mmap's the file and iterates throught the entire file to generate an index
Sequential access | Scans the buffer for the next byte separator | Quick, N records are read at once and cached 
Random access | O(N), just don't | Quick, O(1) via index
Via reducers | Buffer is divided in half repeatedly until it is smaller than specified size, and then entire buffer is converted to String[] for processing | treated exactly like a Clojure vector, but each thread gets it's own cache.

##### Advice
* If you'll only be reading the entire file at a time, then use ```iota/seq```. 
* If you need random access across the file, then use ```iota/vec```. 
* If you need random access and line numbers, then ```iota/numbered-vec```.
* **WARNING** If you're not using reducers with ```iota/seq```, use Clojure's ```line-seq``` instead! 

##### Note
* for iota/vec and iota/seq, empty lines will return nil.
* for iota/numbered-vec, empty lines will return the line number as a String.


## Usage

```clojure
(def file-vec (iota/vec filename)) ;; Map the file into memory, and generate index of lines. Slow.
(def file-seq (iota/seq filename)) ;; Map the file into memory. Quick.

;; Returns first line of file
(first file-vec) 
(first file-seq)

(last file-vec) ;; Returns last line of file

(nth file-vec 2) ;; Returns the 3rd line of the file

;; Count number of non-empty fields in TSV file
(->> (iota/seq filename)
     (clojure.core.reducers/filter identity) ;; filter out empty lines
     (clojure.core.reducers/map  #(->> (clojure.string/split % #"[\t]" -1)
                                       (filter (fn [^String s] (not (.isEmpty s)))) ;; Remove empty fields
                                       count))
     (clojure.core.reducers/fold +))

;; Skips the first line of the file, good for ignoring a header
(iota/subvec file-vec 1) 
(rest file-seq) 
```


## Known issues;
* Records must be delimited by a single byte value, hence 2 Byte encodings like UTF-16 and UCS-2 can't be parsed correctly.
* If any consecutive record separator bytes (default ASCII newline)
  are separated by more than `Integer/MAX_VALUE` = 2**31-1 =
  2,147,483,647 bytes in the file, probably something wrong will
  happen.  The separation distance will be truncated to fit into a
  32-bit signed Java `int` (perhaps negative), and that number of
  bytes will be used instead, perhaps causing fewer bytes to be
  converted in the returned string, or perhaps causing a
  [`NegativeArraySizeException`](https://docs.oracle.com/javase/8/docs/api/java/lang/NegativeArraySizeException.html)
  to be thrown.
* Each sequence of bytes between record separators will be converted
  to a Java String by treating it as if it is encoded in UTF-8.  This
  might cause an `UnsupportedEncodingException` to be thrown, which
  the iota library will catch internally and discard, with no
  indication to the caller that this happened.  The string returned in
  such cases is undefined.


## Artifacts

Iota artifacts are [available on Clojars](https://clojars.org/iota) with instructions for leiningen, gradle, and maven.

## License

MIT
http://opensource.org/licenses/MIT

I'd also like to thank my employer Gracenote, for allowing me to create this open source port.

Copyright (C) 2012-2013 Alan Busby
