---
layout: post
title: golang里支持有序key的json
---

背景是用的某知名开源软件，提供插件仓库的时候，使用了sha1 sha512和证书对内容进行校验，但实现的方式却有点奇葩，是先对原始json文件的key做排序，生成校验信息后，直接增加一个顶级key为signature的dict，将所有校验信息写在这个dict里。读取的时候，取出这个key=signature的dict，再删除这个dict，再重新计算剩下来的整个json的sha来判断是否相同。

这个方式，在Java的多个json库，反序列化为JSONObject或类似对象后，是可以支持的，python的json库的[dump方法](https://docs.python.org/2.7/library/json.html#json.dump)也有sort_keys参数来支持这个场景，但很不幸的，golang内置的json库并不直接支持这个场景，当然这个需求的[issue](https://github.com/golang/go/issues/27179)是有的，但golang官方对这种需求的态度一向hmmm。。。

golang没办法直接支持的原因是没有JSONObject这样的类型，而golang的map又是key乱序，但如果能克服这两个问题，就同样可以解决。

幸好，issue里是有提到解决方案的思路的，就是用 [Decoder.Token](https://pkg.go.dev/encoding/json#Decoder.Token) 方法手动解析出key然后手动管理，[RawMessage](https://pkg.go.dev/encoding/json#RawMessage) 来临时承载value数据，实现一个简易的LinkedHashMap，需要注意的是，序列化过程不能直接用 [Encoder.Encode](https://pkg.go.dev/encoding/json#Encoder.Encode)，因为这个方法会在每个value结尾[强行写个换行](https://cs.opensource.google/go/go/+/refs/tags/go1.17.7:src/encoding/json/stream.go;l=217)，破坏原有的格式。

```golang
package main

import (
	"bytes"
	"crypto/sha512"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"log"
	"os"
)

type orderedJsonDict struct {
	m    map[string]json.RawMessage
	keys []string
}

func (o *orderedJsonDict) MarshalJSON() ([]byte, error) {
	buf := &bytes.Buffer{}
	buf.WriteByte('{')
	for i, k := range o.keys {
		if i != 0 {
			buf.WriteByte(',')
		}
		buf.WriteByte('"')
		buf.WriteString(k)
		buf.WriteByte('"')
		buf.WriteByte(':')
		buf.Write(o.m[k])
	}
	buf.WriteByte('}')
	return buf.Bytes(), nil
}

func (o *orderedJsonDict) UnmarshalJSON(b []byte) error {
	dec := json.NewDecoder(bytes.NewReader(b))
	const delimLeft, delimRight = json.Delim('{'), json.Delim('}')
	if t, err := dec.Token(); err != nil {
		return fmt.Errorf("decoder.Token: %w", err)
	} else if t != delimLeft {
		return fmt.Errorf("not a JSON object")
	}
	if o.m == nil {
		o.m = make(map[string]json.RawMessage)
	}
	for dec.More() {
		var key string
		if t, err := dec.Token(); err != nil {
			return fmt.Errorf("decoder.Token: %w", err)
		} else if t == delimRight {
			break
		} else {
			var ok bool
			key, ok = t.(string)
			if !ok {
				return fmt.Errorf("not a JSON object")
			}
		}
		var rm json.RawMessage
		if err := dec.Decode(&rm); err != nil {
			return fmt.Errorf("decoder.Decode value: %w", err)
		} else {
			o.m[key] = rm
			o.keys = append(o.keys, key)
		}
	}
	return nil
}

func (o *orderedJsonDict) Get(key string) (json.RawMessage, bool) {
	v, ok := o.m[key]
	return v, ok
}

func (o *orderedJsonDict) Set(key string, value json.RawMessage) {
	o.m[key] = value
	for _, k := range o.keys {
		if k == key {
			return
		}
	}
	o.keys = append(o.keys, key)
}

func (o *orderedJsonDict) Delete(key string) {
	delete(o.m, key)
	for i, k := range o.keys {
		if k == key {
			o.keys = append(o.keys[:i], o.keys[i+1:]...)
			return
		}
	}
}

func main() {
	f, err := os.Open(os.Args[1])
	if err != nil {
		log.Fatal(err)
	}
	defer f.Close()
	jd := &orderedJsonDict{}
	if err := json.NewDecoder(f).Decode(&jd); err != nil {
		log.Fatal(err)
	}

	sigMap := make(map[string]interface{})
	signatures, _ := jd.Get("signature")
	jd.Delete("signature")
	if err := json.Unmarshal(signatures, &sigMap); err != nil {
		log.Fatalf("json.Unmarshal signature error: %v", err)
	}
	want := fmt.Sprint(sigMap["correct_digest512"])
	log.Printf("correct_digest512: %s", want)

	sha := sha512.New()
	if buf, err := jd.MarshalJSON(); err != nil {
		log.Fatalf("json.Marshal error: %v", err)
	} else {
		sha.Write(buf)
	}
	got := hex.EncodeToString(sha.Sum(nil))

	log.Printf("calculated_digest512: %s", got)
}
```