---
layout: post
title: go的7个常见错误
---
1、没有真正理解interface（Not Accepting Interfaces）

Stop Doing This!!
    
    func (page *Page) saveSourceAs(path string) { 
        b := new(bytes.Buffer) 
        b.Write(page.Source.Content) 
        page.saveSource(b.Bytes(), path)
    }
    func (page *Page) saveSource(by []byte, inpath string) { 
        WriteToDisk(inpath, bytes.NewReader(by)) 
    } // Stop Doing This!!

Instead

    func (page *Page) saveSourceAs(path string) { 
        b := new(bytes.Buffer) 
        b.Write(page.Source.Content) 
        page.saveSource(b, path) 
    } 
    func (page *Page) saveSource(b io.Reader, inpath string) { 
        WriteToDisk(inpath, b) 
    } // Instead
    
2、没有正确使用Io.Reader & Io.Writer（Not Using Io.Reader & Io.Writer）
Io.Reader & Io.Writer 

1) Simple & ﬂexible interfaces for many operations around input and output

2) Provides access to a huge wealth of functionality 

3)Keeps operations extensible

    type Reader interface { 
        Read(p []byte) (n int, err error) 
    } 
    type Writer interface { 
        Write(p []byte) (n int, err error) 
    }
Really stop do this!

    func (v *Viper) ReadBufConfig(buf *bytes.Buffer) error { 
        v.config = make(map[string]interface{}) 
        v.marshalReader(buf, v.config) return nil 
    }
Instead

    func (v *Viper) ReadConfig(in io.Reader) error { 
        v.config = make(map[string]interface{}) 
        v.marshalReader(in, v.config) 
        return nil 
    }
3、接口滥用（Requiring Broad Interfaces）

1）Interfaces Are Composable

2）Functions should only accept interfaces that require the methods they need

3）Functions should not accept a broad interface when a narrow one would work

4）Compose broad interfaces made from narrower ones

    type File interface { 
        io.Closer 
        io.Reader 
        io.ReaderAt 
        io.Seeker 
        io.Writer 
        io.WriterAt 
    } // Composing Interfaces

Requiring Broad Interfaces

    func ReadIn(f File) { 
        b := []byte{} n, err := f.Read(b) 
        … 
    }
    
Requiring Narrow Interfaces

    func ReadIn(r Reader) { 
        b := []byte{} n, err := r.Read(b) 
        … 
    }
