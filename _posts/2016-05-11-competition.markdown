---
layout: post
title:  "Systems Design Review"
date:   2016-05-11 22:26:33 -0400
published: true
draft: true
permalink: /:title
---



**PROMPT:** 
You are using an in Key-Value Store (KVS) to store file data. 
The KVS can only store binary data up to 2MB, write a wrapper around the KVS API (SET, GET) so that the wrapper can store files larger than 2MB.


**Solution:**
 * Wrapper's Public API be comprised of SET and GET.
 * Wrapper will chunk files >2MB and store them using an internal key map. 
 * Multiple clients will need to be able to use wrapper
 * Each 2MB chunk will have header data that points to the next Chunk (Linked List style)
 * Set() will throw error user provided key violates internal chunk keys
 * StreamFile in 2MB chunks

```

import fs
import buffer


class wrapper {
	constructor(kvs_connection){
		this.kvs = kvs_connection
	}
	_fileStream(path){
		return fs.createReadStream(path)
	}
	set(key, pathToFile){
		if ('@@' in key){
			throw new Error('@@ is used internally to namespace keys. ')
		}
		let chunkCounter = 0;
		const stream = this._fileStream(pathToFile);
		let count = 0;
		let acc = Buffer.alloc(1800000);
		let	chunk = buffer.alloc(256)
		stream.on('data', function(chunk){
			acc += chunk.length
			acc.write(chunk.data)
			if (acc > 1800000){
				stream.pause();
				chunk.write('@@' + key + chunkCounter) 
				this.kvs.set(key, chunk + acc)
				acc = Buffer.alloc(1800000)
				chunk = buffer.alloc(256)
				chunkCounter+= 1
				count = 0
				key = '@@' + key + chunkCounter
				stream.resume()
			}
		})
		stream.on('end', function(){
			let chunk = Buffer.from('@@' + key + chunkCounter) 
			this.kvs.set(key, chunk+acc)
		})
	}	
	get(key){
		let document = Buffer()
		while (key){
			let documentChunk = Buffer.from(this.kvs.get(key))
			key = documentChunk.read(0, 256, 'utf').trim()
			document+= documentChunk.slice(256)
		}
		return document
	}
}


```


Concerns:
Key Collision
