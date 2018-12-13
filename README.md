# A Guide to Modern OpenGL Functions

## Index

* [DSA](#dsa-direct-state-access)
  * [glTexture](#gltexture)
    * [glCreateTextures](#glcreatetextures)
    * [glTextureParameter](#gltextureparameter)
    * [glTextureStorage](#gltexturestorage)
    * [glBindTextureUnit](#glbindtextureunit)
    * [Generating Mip Maps](#generating-mip-maps)
    * [Uploading Cube Maps](#uploading-cube-maps)
    * [Where is glTextureImage?](#where-is-gltextureimage)

  * [glFramebuffer](#glframebuffer)
    * [glCreateFramebuffers](#glcreateframebuffers)
    * [glBlitNamedFramebuffer](#glblitnamedframebuffer)
    * [glClearNamedFramebuffer](#glclearnamedframebuffer)

  * [glBuffer](#glbuffer)
    * [glCreateBuffers](#glcreatebuffers)
    * [glNamedBufferData](#glnamedbufferdata)
    * [glVertexAttribFormat & glBindVertexBuffer](#glvertexattribformat--glbindvertexbuffer)

* [Detailed Messages with Debug Output](#detailed-messages-with-debug-output)
* [Storing Index and Vertex Data Under Single Buffer](#storing-index-and-vertex-data-under-single-buffer)
* [Ideal Way Of Retrieving All Uniform Names](#ideal-way-of-retrieving-all-uniform-names)
* [Texture Atlases vs Arrays](#texture-atlases-vs-arrays)
* [Texture Views & Aliases](#texture-views--aliases)
* [Setting up Mix & Match Shaders with Program Pipelines](#setting-up-mix--match-shaders-with-program-pipelines)
* [Faster Reads and Writes with Persistent Mapping](#faster-reads-and-writes-with-persistent-mapping)
* [More Information](#more-information)

What this is:

* A guide on how to apply modern OpenGL functionality.

What this is not:

* A guide on modern OpenGL rendering techniques.

When I say modern I'm talking DSA modern, not VAO modern, because that's old modern or "middle" GL (however I will be covering some from it), I can't tell you what minimal version you need to make use of DSA because it's not clear at all but you can check if you support it yourself with something like glew's `glewIsSupported("ARB_direct_state_access")` or checking your API version.

## DSA (Direct State Access)
With DSA we, in theory, can keep our bind count outside of drawing operations at zero. Great right? Sure, but if you were to research how to use all the new DSA functions you'd have a hard time finding anywhere where it's all explained, which is what this guide is all about.

###### DSA Naming Convention

The [wiki page](https://www.opengl.org/wiki/Direct_State_Access) does a fine job comparing the DSA naming convention to the traditional one so I stole their table:

| OpenGL Object Type | Context Object Name | DSA Object Name  |
| :------------- |:-------------| :-----|
| [Texture Object](https://www.opengl.org/wiki/Texture) | Tex | Texture |
| [Framebuffer Object](https://www.opengl.org/wiki/Framebuffer_Object) | Framebuffer | NamedFramebuffer |
| [Buffer Object](https://www.opengl.org/wiki/Buffer_Object) | Buffer | NamedBuffer |
| [Transform Feedback Object](https://www.opengl.org/wiki/Transform_Feedback_Object) | TransformFeedback | TransformFeedback |
| [Vertex Array Object](https://www.opengl.org/wiki/Vertex_Array_Object) | N/A | VertexArray |
| [Sampler Object](https://www.opengl.org/wiki/Sampler_Object) | N/A | Sampler |
| [Query Object](https://www.opengl.org/wiki/Query_Object) | N/A | Query |
| [Program Object](https://www.opengl.org/wiki/Program_Object) | N/A | Program |

### glTexture
------
* The texture related calls aren't hard to figure out so let's jump right in.

###### glCreateTextures
* [`glCreateTextures`](http://docs.gl/gl4/glCreateTextures) is the equivalent of [`glGenTextures`](http://docs.gl/gl4/glGenTextures) + [`glBindTexture`](http://docs.gl/gl4/glBindTexture)(for initialization).

```c
void glCreateTextures(GLenum target, GLsizei n, GLuint *textures);
```

Unlike [`glGenTextures`](http://docs.gl/gl4/glGenTextures) [`glCreateTextures`](http://docs.gl/gl4/glCreateTextures) will create the handle *and* initialize the object which is why the field `GLenum target`  is listed as the internal initialization depends on knowing the type.

So this:
```c
glGenTextures(1, &name);
glBindTexture(GL_TEXTURE_2D, name);
```
DSA-ified becomes:
```c
glCreateTextures(GL_TEXTURE_2D, 1, &name);
```

###### glTextureParameter

* [`glTextureParameter`](http://docs.gl/gl4/glTexParameter) is the equivalent of [`glTexParameterX`](http://docs.gl/gl4/glTexParameter)

```c
void glTextureParameteri(GLuint texture, GLenum pname, GLenum param);
```

There isn't much to say about this family of functions; they're used exactly the same but take in the texture name rather than the texture target.

```c
glTextureParameteri(name, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
```

###### glTextureStorage

* [`glTextureStorage`](http://docs.gl/gl4/glTexStorage2D) is semi-equivalent to [`glTexStorage`](http://docs.gl/gl4/glTexStorage2D) ([Where is glTextureImage?](#where-is-gltextureimage)).

The [`glTextureStorage`](http://docs.gl/gl4/glTexStorage2D) and [`glTextureSubImage`](http://docs.gl/gl4/glTexSubImage2D) families are the same exact way.

Time for the big comparison:

```c
glGenTextures(1, &name);
glBindTexture(GL_TEXTURE_2D, name);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

glTexStorage2D(GL_TEXTURE_2D, 1, GL_RGBA8, width, height);
glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
```

```c
glCreateTextures(GL_TEXTURE_2D, 1, &name);

glTextureParameteri(name, GL_TEXTURE_WRAP_S, GL_CLAMP);
glTextureParameteri(name, GL_TEXTURE_WRAP_T, GL_CLAMP);
glTextureParameteri(name, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTextureParameteri(name, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

glTextureStorage2D(name, 1, GL_RGBA8, width, height);
glTextureSubImage2D(name, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
```

###### glBindTextureUnit

* [`glBindTextureUnit`](http://docs.gl/gl4/glBindTextureUnit) is the equivalent of [`glActiveTexture`](http://docs.gl/gl4/glActiveTexture) + [`glBindTexture`](http://docs.gl/gl4/glBindTexture)

Defeats the need for:
```c
glActiveTexture(GL_TEXTURE0 + 3);
glBindTexture(GL_TEXTURE_2D, name);
```

And replaces it with a simple:
```c
glBindTextureUnit(3, name);
```

###### Generating Mip Maps

* [`glGenerateTextureMipmap`](http://docs.gl/gl4/glGenerateMipmap) is the equivalent of [`glGenerateMipmap`](http://docs.gl/gl4/glGenerateMipmap)

Takes in the texture name instead of the texture target.

```c
void glGenerateTextureMipmap(GLuint texture);
```

###### Uploading Cube Maps

I should briefly point out that in order to upload cube map textures you need to use [`glTextureSubImage3D`](http://docs.gl/gl4/glTexSubImage3D).

```c
glTextureStorage2D(name, 1, GL_RGBA8, bitmap.width, bitmap.height);

for (size_t face = 0; face < 6; ++face)
{
	auto const& bitmap = bitmaps[face];
	glTextureSubImage3D(name, 0, 0, 0, face, bitmap.width, bitmap.height, 1, bitmap.format, GL_UNSIGNED_BYTE, bitmap.pixels);
}
```

###### Where is glTextureImage?

If you look at the OpenGL function listing you will see a lack of `glTextureImage` and here's why:

`glTexImage` left a lot to be desired, it's very easy to end up with invalid textures because the default filtering requires several mipmap levels to be present (`GL_NEAREST_MIPMAP_LINEAR` and a `GL_TEXTURE_MAX_LEVEL` of `1000`) with all the mipmap storage needing to be specified individually, you could even give them a bunch of inconsistent sizes and the driver would only check for consistency at draw time.

The answer to this was [`glTexStorage`](http://docs.gl/gl4/glTexStorage2D), when it came time for DSA they left the replaced `glTexImage` in the dust.

Storage provides a way to create complete textures with checks done on-call, which means less room for error, it solves most if not all problems brought on by mutable textures. 

tl;dr "Immutable textures are a more robust approach to handle textures"

* However be mindful as allocating immutable textures requires physical video memory to be available upfront rather than having the driver deal with when and where the data goes, this means it's very possible to unintentionally exceed your card's capacity. 

Sources: [ARB_texture_storage](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_texture_storage.txt), ["What does glTexStorage do?"](https://stackoverflow.com/questions/9224300/what-does-gltexstorage-do), ["What's the DSA version of glTexImage2D?"](https://gamedev.stackexchange.com/questions/134177/whats-the-dsa-version-of-glteximage2d)

### glFramebuffer
------
###### glCreateFramebuffers

* [`glCreateFramebuffers`](http://docs.gl/gl4/glCreateFramebuffers) is the equivalent of [`glGenFramebuffers`](http://docs.gl/gl4/glGenFramebuffers)

[`glCreateFramebuffers`](http://docs.gl/gl4/glCreateFramebuffers) is used exactly the same but initializes the object for you.

Everything else is pretty much the same but takes in the framebuffer handle instead of the target.

```cpp
glCreateFramebuffers(1, &fbo);

glNamedFramebufferTexture(fbo, GL_COLOR_ATTACHMENT0, tex, 0);
glNamedFramebufferTexture(fbo, GL_DEPTH_ATTACHMENT, depthTex, 0);

if(glCheckNamedFramebufferStatus(fbo, GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
	std::cerr << "framebuffer error\n";
```

###### glBlitNamedFramebuffer

* [`glBlitNamedFramebuffer`](http://docs.gl/gl4/glBlitFramebuffer) is the equivalent of [`glBlitFramebuffer`](http://docs.gl/gl4/glBlitFramebuffer)

The difference here is that we no longer need to bind the two framebuffers and specify which is which through the `GL_READ_FRAMEBUFFER` and `GL_WRITE_FRAMEBUFFER` enums.

```c
glBindFramebuffer(GL_READ_FRAMEBUFFER, fbo_src);
glBindFramebuffer(GL_DRAW_FRAMEBUFFER, fbo_dst);

glBlitFramebuffer(src_x, src_y, src_w, src_h, dst_x, dst_y, dst_w, dst_h, GL_COLOR_BUFFER_BIT, GL_LINEAR);
```
Becomes
```c
glBlitNamedFramebuffer(fbo_src, fbo_dst, src_x, src_y, src_w, src_h, dst_x, dst_y, dst_w, dst_h, GL_COLOR_BUFFER_BIT, GL_LINEAR);
```

###### glClearNamedFramebuffer

* [`glClearNamedFramebuffer`](http://docs.gl/gl4/glClearBuffer) is the equivalent of [`glClearBuffer`](http://docs.gl/gl4/glClearBuffer)

There are two ways to go about clearing a framebuffer:

The most familar way
```c
glBindFramebuffer(fb);
glClearColor(r, g, b, a);
glClearDepth(d);
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

and the more versatile per-attachment way
```c
glBindFramebuffer(fb);
glClearBufferfv(GL_COLOR, col_buff_index, &rgba);
glClearBufferfv(GL_DEPTH, 0, &d);
```

`col_buff_index` is the attachment index, so it would be equivalent to `GL_DRAW_BUFFER0 + col_buff_index`, and the draw buffer index for depth is always `0`.

As you can see with `glClearBuffer` we can clear the texels of any attachment to some value, both methods are similar enough that you could reimplement the functions of method 1 using those of method 2.

Despite the name it has nothing to do with buffer objects and this gets cleared up with the DSA version: `glClearNamedFramebuffer` 

So the DSA version looks like this:
```c
glClearNamedFramebufferfv(fb, GL_COLOR, col_buff_index, &rgba);
glClearNamedFramebufferfv(fb, GL_DEPTH, 0, &d);
```

`fb` can be `0` if you're clearing the default framebuffer.

### glBuffer
------
None of the DSA glBuffer functions ask for the buffer target and is only required to be specified whilst drawing.

###### glCreateBuffers
* [`glCreateBuffers`](glCreateBuffers) is the equivalent of [`glGenBuffers`](http://docs.gl/gl4/glGenBuffers) + [`glBindBuffer`](http://docs.gl/gl4/glBindBuffer)(the initialization part)

[`glCreateBuffers`](http://docs.gl/gl4/glGenBuffers) is used exactly like its traditional equivalent and automatically initializes the object.

###### glNamedBufferData
* [`glNamedBufferData`](http://docs.gl/gl4/glBufferData) is the equivalent of [`glBufferData`](http://docs.gl/gl4/glBufferData)

[`glNamedBufferData`](http://docs.gl/gl4/glBufferData) is just like [`glBufferData`](http://docs.gl/gl4/glBufferData) but instead of requiring the buffer target it takes in the buffer handle itself.

###### glVertexAttribFormat & glBindVertexBuffer
* [`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) and [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) are the equivalent of [`glVertexAttribPointer`](http://docs.gl/gl4/glVertexAttribPointer)

If you aren't familiar with the application of [`glVertexAttribPointer`](http://docs.gl/gl4/glVertexAttribPointer) it is used like so:

```c
struct vertex_t { vec3 pos, nrm; vec2 tex; };

glBindVertexArray(vao);
glBindBuffer(GL_ARRAY_BUFFER, vbo);

glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, pos));
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, nrm));
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, tex));
```

[`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) isn't much different, the main thing with it is that it's one out of a two-parter with [`glVertexAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding).

In order to get out the same effect as the previous snippet we first need to make a call to [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer). Despite *Bind* being in the name it isn't the same kind as for instance: `glBindTexture`.

Here's how they're both put into action:

```c
struct vertex_t { vec3 pos, nrm; vec2 tex; };

glBindVertexArray(vao);
glBindVertexBuffer(0, vbo, 0, sizeof(vertex_t));

glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glVertexAttribFormat(0, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, pos));
glVertexAttribFormat(1, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, nrm));
glVertexAttribFormat(2, 2, GL_FLOAT, GL_FALSE, offsetof(vertex_t, tex));

glVertexAttribBinding(0, 0);
glVertexAttribBinding(1, 0);
glVertexAttribBinding(2, 0);
```

Although this is the newer way of going about it this isn't fully DSA as we still need to make that VAO bind call, to go all the way we need to transform [`glEnableVertexAttribArray`](http://docs.gl/gl4/glEnableVertexAttribArray), [`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat), [`glVertexAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding), and [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) into [`glEnableVertexArrayAttrib`](http://docs.gl/gl4/glEnableVertexAttribArray), [`glVertexArrayAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat), [`glVertexArrayAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding), and [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer).

```c
glVertexArrayVertexBuffer(vao, 0, data->vbo, 0, sizeof(vertex_t));

glEnableVertexArrayAttrib(vao, 0);
glEnableVertexArrayAttrib(vao, 1);
glEnableVertexArrayAttrib(vao, 2);

glVertexArrayAttribFormat(vao, 0, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, pos));
glVertexArrayAttribFormat(vao, 1, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, nrm));
glVertexArrayAttribFormat(vao, 2, 2, GL_FLOAT, GL_FALSE, offsetof(vertex_t, tex));

glVertexArrayAttribBinding(vao, 0, 0);
glVertexArrayAttribBinding(vao, 1, 0);
glVertexArrayAttribBinding(vao, 2, 0);
```

The version that takes in the VAO for binding the VBO, [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer), has an equivalent for the IBO: [`glVertexArrayElementBuffer`](http://docs.gl/gl4/glVertexArrayElementBuffer).

All together this is how uploading an indexed model with *only* DSA should look:

```c
glCreateBuffers(1, &vbo);	
glNamedBufferStorage(vbo, sizeof(vertex_t)*vertex_count, vertices, GL_DYNAMIC_STORAGE_BIT);

glCreateBuffers(1, &ibo);
glNamedBufferStorage(ibo, sizeof(uint32_t)*index_count, indices, GL_DYNAMIC_STORAGE_BIT);

glCreateVertexArrays(1, &vao);

glVertexArrayVertexBuffer(vao, 0, vbo, 0, sizeof(vertex_t));
glVertexArrayElementBuffer(vao, ibo);

glEnableVertexArrayAttrib(vao, 0);
glEnableVertexArrayAttrib(vao, 1);
glEnableVertexArrayAttrib(vao, 2);

glVertexArrayAttribFormat(vao, 0, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, pos));
glVertexArrayAttribFormat(vao, 1, 3, GL_FLOAT, GL_FALSE, offsetof(vertex_t, nrm));
glVertexArrayAttribFormat(vao, 2, 2, GL_FLOAT, GL_FALSE, offsetof(vertex_t, tex));

glVertexArrayAttribBinding(vao, 0, 0);
glVertexArrayAttribBinding(vao, 1, 0);
glVertexArrayAttribBinding(vao, 2, 0);
```

## Detailed Messages with Debug Output

[`KHR_debug`](http://www.opengl.org/registry/specs/KHR/debug.txt) has been in core since version 4.3 and it's a big step up from how we used to do error polling.

With [Debug Output](https://www.khronos.org/opengl/wiki/Debug_Output) we can receive meaningful messages on the state of the GL through a callback function that we'll be providing.

All it takes to get running are two calls: [`glEnable`](http://docs.gl/gl4/glEnable) & [`glDebugMessageCallback`](http://docs.gl/gl4/glDebugMessageCallback).
```cpp
glEnable(GL_DEBUG_OUTPUT);
glDebugMessageCallback(message_callback, nullptr);
```

Your callback must have the signature `void callback(GLenum src, GLenum type, GLuint id, GLenum severity, GLsizei length, GLchar const* msg, void const* user_param)`. 

Here's how I have mine defined:

```cpp
void message_callback(GLenum source, GLenum type, GLuint id, GLenum severity, GLsizei length, GLchar const* message, void const* user_param)
{
	auto const src_str = [source]() {
		switch (source)
		{
		case GL_DEBUG_SOURCE_API: return "API";
		case GL_DEBUG_SOURCE_WINDOW_SYSTEM: return "WINDOW SYSTEM";
		case GL_DEBUG_SOURCE_SHADER_COMPILER: return "SHADER COMPILER";
		case GL_DEBUG_SOURCE_THIRD_PARTY: return "THIRD PARTY";
		case GL_DEBUG_SOURCE_APPLICATION: return "APPLICATION";
		case GL_DEBUG_SOURCE_OTHER: return "OTHER";
		}
	}();

	auto const type_str = [type]() {
		switch (type)
		{
		case GL_DEBUG_TYPE_ERROR: return "ERROR";
		case GL_DEBUG_TYPE_DEPRECATED_BEHAVIOR: return "DEPRECATED_BEHAVIOR";
		case GL_DEBUG_TYPE_UNDEFINED_BEHAVIOR: return "UNDEFINED_BEHAVIOR";
		case GL_DEBUG_TYPE_PORTABILITY: return "PORTABILITY";
		case GL_DEBUG_TYPE_PERFORMANCE: return "PERFORMANCE";
		case GL_DEBUG_TYPE_MARKER: return "MARKER";
		case GL_DEBUG_TYPE_OTHER: return "OTHER";
		}
	}();

	auto const severity_str = [severity]() {
		switch (severity) {
		case GL_DEBUG_SEVERITY_NOTIFICATION: return "NOTIFICATION";
		case GL_DEBUG_SEVERITY_LOW: return "LOW";
		case GL_DEBUG_SEVERITY_MEDIUM: return "MEDIUM";
		case GL_DEBUG_SEVERITY_HIGH: return "HIGH";
		}
	}();

	std::cout << src_str << ", " << type_str << ", " << severity_str << ", " << id << ": " << message << '\n';
}
```

There will be times when you want to filter your messages, maybe you're interested in anything but notifications. OpenGL has a function for this: [`glDebugMessageControl`](http://docs.gl/gl4/glDebugMessageControl).

Here's how we use it to disable notifications:

```cpp
glDebugMessageControl(GL_DONT_CARE, GL_DONT_CARE, GL_DEBUG_SEVERITY_NOTIFICATION, 0, nullptr, GL_FALSE);
```

Something we can do is have messages fire synchronously where it will call on the same thread as the context and from within the OpenGL call. This way we can gaurantee function call order and this means if we were to add a breakpoint into the definition of our callback we could traverse the call stack and locate the origin of the error.

All it takes is another call to [`glEnable`](http://docs.gl/gl4/glEnable) with the value of `GL_DEBUG_OUTPUT_SYNCHRONOUS`, so you end up with this:

```cpp
glEnable(GL_DEBUG_OUTPUT);
glEnable(GL_DEBUG_OUTPUT_SYNCHRONOUS);
glDebugMessageCallback(message_callback, nullptr);
```

Farewell, `glGetError`.

## Storing Index and Vertex Data Under Single Buffer

Any material on OpenGL separates vertex and index data between buffers, this is because the [vertex_buffer_object](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_vertex_buffer_object.txt) spec strongly urges to do so, the reasoning for this is that different GL implementations may have different memory type requirements, so having the index data in its own buffer allows the driver to decide the optimal storage strategy.

This was useful when there were were several ways to attach GPUs to the main system, technically there still are, but AGP was completely phased out by PCIe about a decade ago and regular PCI ports aren't really used for this anymore save a few cases.

So managing two buffers for indexed geometry is no longer advantageous.

All we need to do is store the indices before the vertex data and tell OpenGL where the vertices begin, this is achieved with [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer)'s offset parameter.

```cpp
GLint alignment = GL_NONE;
glGetIntegerv(GL_UNIFORM_BUFFER_OFFSET_ALIGNMENT, &alignment);

GLuint vao 	= GL_NONE;
GLuint buffer 	= GL_NONE;

auto const ind_len = GLsizei(ind_buffer.size() * sizeof(element_t));
auto const vrt_len = GLsizei(vrt_buffer.size() * sizeof(vertex_t));

auto const ind_len_aligned = align(ind_len, alignment);
auto const vrt_len_aligned = align(vrt_len, alignment);

glCreateBuffers(1, &buffer);
glNamedBufferStorage(buffer, ind_len_aligned + vrt_len_aligned, nullptr, GL_DYNAMIC_STORAGE_BIT);

glNamedBufferSubData(buffer, 0		    , ind_len, ind_buffer.data());
glNamedBufferSubData(buffer, ind_len_aligned, vrt_len, vrt_buffer.data());

glCreateVertexArrays(1, &vao);
glVertexArrayVertexBuffer(vao, 0, buffer, ind_len_aligned, sizeof(vertex_t));
glVertexArrayElementBuffer(vao, buffer);

//continue with setup
```

## Ideal Way Of Retrieving All Uniform Names

There is material out there that teach beginners to retrieve uniform information by manually parsing the shader source strings, please don't do this.

Here is how it should be done:

```cpp
struct uniform_info_t
{ 
	GLint location;
	GLsizei count;
};

GLint uniform_count = 0;
glGetProgramiv(program_name, GL_ACTIVE_UNIFORMS, &uniform_count);

if (uniform_count != 0)
{
	GLint 	max_name_len = 0;
	GLsizei length = 0;
	GLsizei count = 0;
	GLenum 	type = GL_NONE;
	glGetProgramiv(program_name, GL_ACTIVE_UNIFORM_MAX_LENGTH, &max_name_len);
	
	auto uniform_name = std::make_unique<char[]>(max_name_len);

	std::unordered_map<std::string, uniform_info_t> uniforms;

	for (GLint i = 0; i < uniform_count; ++i)
	{
		glGetActiveUniform(program_name, i, max_name_len, &length, &count, &type, uniform_name.get());

		uniform_info_t uniform_info = {};
		uniform_info.location = glGetUniformLocation(program_name, uniform_name.get());
		uniform_info.count = count;

		uniforms.emplace(std::make_pair(std::string(uniform_name.get(), length), uniform_info));
	}
}
```

Note that the `GLsizei size` parameter refers to the number of locations the uniform takes up with `mat3`, `vec4`, `float`, etc. being 1 and arrays having it be the number of elements, the locations are arranged in a way that allows you to do `array_location + element_number` to find the location of an element, so if you wanted to write to element `5` it would be done like this:
```cpp
glProgramUniformXX(program_name, uniforms["my_array[0]"].location + 5, value);
``` 
or if you want to modify the whole array:
```cpp
glProgramUniformXXv(program_name, uniforms["my_array[0]"].location, uniforms["my_array[0]"].count, my_array);
```
*Ideally UBOs would be used when dealing with collections of data **larger than 16K** as it may be slower than packing the data into vec4s and using [`glProgramUniform4f`](http://docs.gl/gl4/glProgramUniform).*

With this you can store the uniform datatype and check it within your uniform update functions.

## Texture Atlases vs Arrays

Array textures are a great way of managing collections of textures of the same size and format. They allow for using a set of textures without having to bind between them.

They turn out to be good as an alternative to atlases as long as some criteria are met, that being all the sub-textures, or swatches, fit under the same dimensions and levels.

The advantages of using this over an atlas is that each layer is treated as a separate texture in terms of wrapping and mipmapping.

Array textures come with three targets: `GL_TEXTURE_1D_ARRAY`, `GL_TEXTURE_2D_ARRAY`, and `GL_TEXTURE_CUBE_MAP_ARRAY`.

2D array textures and 3d textures are similar but are semantically different, the differences come in where the mipmap level are and how layers are filtered.

There is no built-in filtering for interpolation between layers where the Z part of a 3d texture will have filtering available. The same goes for 1d arrays and 2d textures.

2d array
```
|layer 0 	|
|	level 1	|
|	level 2	|
|layer 1 	|
|	level 1	|
|	level 2	|
...
```

3d texture
```
|z off 0 	|
|z off 1 	|
|z off 2 	|
...
|	level 1 |
|	level 2 |
...
```

To allocate a 2D texture array we do this:

```c
GLuint texarray = 0;
GLsizei width = 512, height = 512, layers = 3;
glCreateTextures(GL_TEXTURE_2D_ARRAY, 1, &texarray);
glTextureStorage3D(texarray, 0, GL_RGBA8, width, height, layers);
```

[`glTextureStorage3D`](http://docs.gl/gl4/glTexStorage3D) has been modified to accommodate 2d array textures which I imagine is confusing at first but there's a pattern: the last dimension parameter acts as the layer specifier, so if you were to allocate a 1D texture array you would have to use [`glTextureStorage2D`](http://docs.gl/gl4/glTexStorage2D) with height as the layer capacity.

Anyway, uploading to individual layers is very straightforward:

```c
glTextureSubImage3D(texarray, mipmap_level, offset.x, offset.y, layer, width, height, 1, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
```

It's super duper simple.

The most notable difference between arrays and atlases in terms of implementation lies in the shader.

To bind a texture array to the context you need a specialized sampler called `samplerXXArray`. We will also need a uniform to store the layer id.

```glsl
#version 450 core

layout (location = 0) out vec4 color;
layout (location = 0) in vec2 tex0;

uniform sampler2DArray texarray;
uniform uint diffuse_layer;

float layer2coord(uint capacity, uint layer)
{
	return max(0, min(float(capacity - 1), floor(float(layer) + 0.5)));
}

void main()
{
	color = texture(texarray, vec3(tex0, layer2coord(3, diffuse_layer)));
}
```

Ideally you should calculate the layer coordinate outside of the shader.

You can take this way further and set up a little UBO/SSBO system of arrays containing layer id and texture array id pairs and update which layer id is used with regular uniforms.

Also, I advise against using ubos and ssbos for per object/draw stuff without a plan otherwise you will end up with everything not working as you'd like because the command queue has no involvement during the reads and writes.

As a bonus let me tell you an easy way to populate a texture array with parts of an atlas, in our case a basic rpg tileset.

Modern OpenGL comes with two generic memory copy functions: [`glCopyImageSubData`](http://docs.gl/gl4/glCopyImageSubData) and [`glCopyBufferSubData`](http://docs.gl/gl4/glCopyBufferSubData). Here we'll be dealing with [`glCopyImageSubData`](http://docs.gl/gl4/glCopyImageSubData), this function allows us to copy sections of a source image to a region of a destination image.
We're going to take advantage of its offset and size parameters so that we can copy tiles from every location and paste them in the appropriate layers within our texture array.

Image files loaded with [nothings](https://github.com/nothings)' [stb_image](https://github.com/nothings/stb) header library.

Here it is:
```cpp
GLsizei image_w, image_h, c, tile_w = 16, tile_h = 16;
stbi_uc* pixels = stbi_load(".\\textures\\tiles_packed.png", &image_w, &image_h, &c, STBI_rgb_alpha);
GLuint tileset;
GLsizei
	tiles_x = (image_w / tile_w),
	tiles_y = (image_h / tile_h),
	tile_count = tiles_x * tiles_y;

glCreateTextures(GL_TEXTURE_2D_ARRAY, 1, &tileset);
glTextureStorage3D(tileset, 1, GL_RGBA8, tile_w, tile_h, tile_count);

{
	GLuint temp_tex = 0;
	glCreateTextures(GL_TEXTURE_2D, 1, &temp_tex);
	glTextureStorage2D(temp_tex, 1, GL_RGBA8, image_w, image_h);
	glTextureSubImage2D(temp_tex, 0, 0, 0, image_w, image_h, GL_RGBA, GL_UNSIGNED_BYTE, pixels);

	for (GLsizei i = 0; i < tile_count; ++i)
	{
		GLint x = (i % tiles_x) * tile_w, y = (i / tiles_x) * tile_h;
		glCopyImageSubData(temp_tex, GL_TEXTURE_2D, 0, x, y, 0, tileset, GL_TEXTURE_2D_ARRAY, 0, 0, 0, i, tile_w, tile_h, 1);
	}
	glDeleteTextures(1, &temp_tex);
}

stbi_image_free(pixels);
```

## Texture Views & Aliases

Texture views allow us to share a section of a texture's storage with an object of a different texture target and/or format. *Share* as in there's no copying of the texture data, any changes you make to a view's data is visible to all views that share the storage along with the original texture object.

Views are mostly indistinguishable from regular texture objects so you can use them as if they are.

The original storage is only freed once all references to it are deleted, if you are familiar with C++'s `std::shared_ptr` it's very similar. 

We can only make views if we use a target and format which is compatible with our original texture, you can read the format tables on the [wiki](https://www.khronos.org/opengl/wiki/Texture_Storage#View_texture_aliases).

Making the view itself is simple, it's only two function calls: [`glGenTextures`](http://docs.gl/gl4/glGenTextures) & [`glTextureView`](http://docs.gl/gl4/glTextureView).

Despite this being about modern OpenGL [`glGenTextures`](http://docs.gl/gl4/glGenTextures) has an important role here: we need only an available texture name and nothing more; this needs to be a completely empty uninitialized object and because this function only handles the generation of a valid handle it's perfect for this.

[`glTextureView`](http://docs.gl/gl4/glTextureView) is how we'll create the view.

If we were to have a typical 2d texture array and needed a view of layer 5 in isolation this is how it would look:

```c
glGenTextures(1, &view_name);
glTextureView(view_name, GL_TEXTURE_2D, src_name, internal_format, min_level, level_count, 5, 1);
```

With this you can bind layer 5 alone as a `GL_TEXTURE_2D` texture.

This is the exact same when dealing with cube maps, the layer parameters will correspond to the cube faces with the layer params of cube map arrays being `cubemap_layer * 6 + face`.

Texture views can be of other views as well, so there could be a texture array, and a view of a section of that array, and another view of a specific layer within that array view. The parameters are relative to the properties of the source. 

The fact that we can specify which mipmaps we want in the view means that we can have views which are just of those specific mipmap levels, so for example you could make textures views of the *N*th mipmap level of a bunch of textures and use only those for expensive texture dependant lighting calculations.

[ARB_texture_view](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_texture_view.txt)

## Setting up Mix & Match Shaders with Program Pipelines

* Nvidia drivers have spotty performance as they lean towards monolithic shader programs, so it may be better suited for non-performance-critical applications.

[Program Pipeline](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_separate_shader_objects.txt) objects allow us to change shader stages on the fly without having to relink them.

To create and set up a simple program pipeline without any debugging looks like this:

```cpp
const char*
	vs_source = load_file(".\\main_shader.vs").c_str(),
	fs_source = load_file(".\\main_shader.fs").c_str();
	
GLuint 
	vs = glCreateShaderProgramv(GL_VERTEX_SHADER, 1, &vs_source),
	fs = glCreateShaderProgramv(GL_FRAGMENT_SHADER, 1, &fs_source),
	pr = 0;
		
glCreateProgramPipelines(1, &pr);
glUseProgramStages(pr, GL_VERTEX_SHADER_BIT, vs);
glUseProgramStages(pr, GL_FRAGMENT_SHADER_BIT, fs);

glBindProgramPipeline(pr);
```

[`glCreateProgramPipelines`](http://docs.gl/gl4/glCreateProgramPipelines) generates the handle and initializes the object, [`glCreateShaderProgramv`](http://docs.gl/gl4/glCreateShaderProgram) generates, initializes, compiles, and links a shader program using the sources given, and [`glUseProgramStages`](http://docs.gl/gl4/glUseProgramStages) attaches the program's stage(s) to the pipeline object. [`glBindProgramPipeline`](http://docs.gl/gl4/glBindProgramPipeline) as you can tell binds the pipeline to the context.

Because our shaders are now looser and flexible we need to get stricter with our input and output variables.
Either we declare the input and output in the same order with the same names or we make their locations explicitly match through the location qualifier.  

I greatly suggest the latter option for non-blocks, this will allow us to set up a well-defined interface while also being flexible with the naming and ordering.
Interface blocks also need to match members.

As collateral for needing a stricter interface we also need to declare the built-in input and output blocks we wish to use for every stage.

The built-in block interfaces are defined as ([from the wiki](https://www.khronos.org/opengl/wiki/Built-in_Variable_(GLSL))):

Vertex:
```glsl
out gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
};
```

Tesselation Control:
```glsl
out gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
} gl_out[];
```

Tesselation Evaluation:
```glsl
out gl_PerVertex {
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
};
```

Geometry:
```glsl

out gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
};
```

An extremely basic vertex shader enabled for use in a pipeline object looks like this:
```glsl
#version 450

out gl_PerVertex { vec4 gl_Position; };

layout (location = 0) in vec3 pos;
layout (location = 1) in vec3 col;

layout (location = 0) out v_out
{
    vec3 col;
} v_out;

void main()
{
    v_out.col = col;
    gl_Position = vec4(pos, 1.0);
}
```

## Faster Reads and Writes with Persistent Mapping

With [`persistent mapping`](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_buffer_storage.txt) we can get a pointer to a region of memory that OpenGL will be using as a sort of intermediate buffer zone, this will allow us to make reads and writes to this area and let the driver decide when to use the contents.

First we need the right flags for both buffer storage creation and the mapping itself:
```cpp
constexpr GLbitfield 
	mapping_flags = GL_MAP_WRITE_BIT | GL_MAP_READ_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT,
	storage_flags = GL_DYNAMIC_STORAGE_BIT | mapping_flags;
```

**GL_MAP_COHERENT_BIT**: 	This flag ensures writes will be seen automagically by the server when done from the client and vice versa.

**GL_MAP_PERSISTENT_BIT**: 	This tells our driver you wish to hold onto the data despite what it's doing.

**GL_MAP_READ_BIT**:	Lets OpenGL know we wish to read from the buffer so that it doesn't freak out when we do.

**GL_MAP_WRITE_BIT**:	Lets OpenGL know we're gonna write to it, if you don't specify this *anything could happen*.

If we don't use these flags for the storage creation GL will reject your mapping request with scorn. What's worse is that you absolutely won't know unless you're doing some form of error checking.

Setting up our immutable storage is very simple:
```cpp
glCreateBuffers(1, &name);
glNamedBufferStorage(name, size, nullptr, storage_flags);
```
Whatever we put in the `const void* data` parameter is arbitrary and marking it as `nullptr` specifies we wish not to copy any data into it.

Here is how we get that pointer we're after:
```cpp
void* ptr = glMapNamedBufferRange(name, offset, size, mapping_flags);
```
You don't have to map it every frame and I advise against it as it harms overall performance.

Make sure to unmap the buffer before deleting it:
```cpp
glUnmapNamedBuffer(name);
glDeleteBuffers(1, &name);
```

If you're using C++ and GSL you can drop it into a span:
```cpp
gsl::span<vertex_t> vertices(reinterpret_cast<vertex_t*>(glMapNamedBufferRange(name, 0, size, mapping_flags)), size);
```

## More information
 * [OpenGL wiki](https://www.khronos.org/opengl/wiki/).
 * [docs.GL](http://docs.gl/)
 * [DSA ARB specification](https://www.opengl.org/registry/specs/ARB/direct_state_access.txt).
##### Have something you would like me to cover and/or fix? Let me know! My Discord is `Fen#0110`.
