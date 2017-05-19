# A Guide to Modern OpenGL Functions

What this is:

 * A guide on how to properly apply scarcely documented modern OpenGL functions.

What this is not:

* A guide on modern OpenGL rendering techniques.

When I say modern I'm talking DSA modern, not VAO modern, because that's old modern (however I will be covering some stuff from around that version), I can't tell you what minimal version you need to make use of DSA because it's not clear at all but you can check if you support it yourself with something like glew's `glewIsSupported("ARB_direct_state_access")`.

## DSA (Direct State Access)
With DSA we, in theory, can keep our bind count outside of drawing operations at zero, great right? Sure, but if you were to research how to use all the new DSA functions you'd have a hard time finding anywhere where it's all explained, which is what this guide is all about.

###### DSA Naming Convention

The [wiki page](https://www.opengl.org/wiki/Direct_State_Access) does a fine job comparing the DSA naming convention to the traditional one but here is the basic table:

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
* The texture related calls aren't complex to figure out so let's jump right in.

###### glCreateTextures
* [`glCreateTextures`](http://docs.gl/gl4/glCreateTextures) equivalent of [`glGenTextures`](http://docs.gl/gl4/glGenTextures) + [`glBindTexture`](http://docs.gl/gl4/glBindTexture)(for initialization).

```c
void glCreateTextures(GLenum target, GLsizei n, GLuint *textures);
```

Unlike [`glGenTextures`](http://docs.gl/gl4/glGenTextures) [`glCreateTextures`](http://docs.gl/gl4/glCreateTextures) will create the handle *and* initialize the object which is why the field `GLenum target`  is listed as the internal initialization depends on knowing the type.

So this:
```c
glGenTextures(1, &id);
glBindTexture(GL_TEXTURE_2D, id);
```
DSA-ified becomes:
```c
glCreateTextures(GL_TEXTURE_2D, 1, &id);
```

###### glTextureParameter

* [`glTextureParameter`](http://docs.gl/gl4/glTexParameter) is the equivalent of [`glTexParameterX`](http://docs.gl/gl4/glTexParameter)

```c
void glTextureParameteri(GLuint texture, GLenum pname, GLenum param);
```

There isn't much to say about this family of functions; they're used exactly the same but take in the texture id rather than the texture target.

```c
glTextureParameteri(id, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
```

The [`glTextureStorage`](http://docs.gl/gl4/glTexStorage2D) and [`glTextureSubImage`](http://docs.gl/gl4/glTexSubImage2D) families are the same exact way.

Time for the big comparison:

```c
glGenTextures(1, &id);
glBindTexture(GL_TEXTURE_2D, id);

glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_CLAMP);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_CLAMP);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

glTexStorage2D(GL_TEXTURE_2D, 1, GL_RGBA8, 512, 512);
glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, 512, 512, GL_RGBA, GL_UNSIGNED_INT, pixels);
```

```c
glCreateTextures(GL_TEXTURE_2D, 1, &id);

glTextureParameteri(id, GL_TEXTURE_WRAP_S, GL_CLAMP);
glTextureParameteri(id, GL_TEXTURE_WRAP_T, GL_CLAMP);
glTextureParameteri(id, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTextureParameteri(id, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

glTextureStorage2D(id, 1, GL_RGBA8, 512, 512);
glTextureSubImage2D(id, 0, 0, 0, 512, 512, GL_RGBA, GL_UNSIGNED_INT, pixels);
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

for (size_t face = 0; face < 6; face++)
{
	const Bitmap& bitmap = bitmaps[face];
	glTextureSubImage3D(name, 0, 0, 0, face, bitmap.width, bitmap.height, 1, bitmap.format, GL_UNSIGNED_INT, bitmap.pixels);
}
```

### glFramebuffer
------
###### glCreateFramebuffers

* [`glCreateFramebuffers`](http://docs.gl/gl4/glCreateFramebuffers) is the equivalent of [`glGenFramebuffers`](http://docs.gl/gl4/glGenFramebuffers)

[`glCreateFramebuffers`](http://docs.gl/gl4/glCreateFramebuffers) is used exactly the same but initializes the object for you.

Everything else is pretty much the same but takes in the framebuffer handle instead of the target.

```c
glCreateFramebuffers(1, &fbo);

glNamedFramebufferTexture(fbo, GL_COLOR_ATTACHMENT0, tex, 0);
glNamedFramebufferTexture(fbo, GL_DEPTH_ATTACHMENT, depthTex, 0);

if(glCheckNamedFramebufferStatus(fbo, GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE)
	printf("framebuffer error");
```


### glBuffer
------
None of the DSA glBuffer functions ask for the buffer target and is only required to be specified whilst drawing.

###### glCreateBuffers
* [`glCreateBuffers`](glCreateBuffers) is the equivalent of [`glGenBuffers`](http://docs.gl/gl4/glGenBuffers) + [`glBindBuffer`](http://docs.gl/gl4/glBindBuffer)(the initialization part)

[`glCreateBuffers`](http://docs.gl/gl4/glGenBuffers) is used exactly like its traditional equivalent and automically initializes the object.

###### glNamedBufferData
* [`glNamedBufferData`](http://docs.gl/gl4/glBufferData) is the equivalent of [`glBufferData`](http://docs.gl/gl4/glBufferData)

[`glNamedBufferData`](http://docs.gl/gl4/glBufferData) is just like [`glGenBuffers`](http://docs.gl/gl4/glGenBuffers) but instead of requiring the buffer target it takes in the buffer handle itself.

###### glVertexAttribFormat & glBindVertexBuffer
* [`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) and [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) are the equivalent of [`glVertexAttribPointer`](http://docs.gl/gl4/glVertexAttribPointer)

If you aren't familiar with the application of [`glVertexAttribPointer`](http://docs.gl/gl4/glVertexAttribPointer) it is used like so:

```c
struct Vertex { vec3 pos, nrm; vec2 tex; };

glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offsetof(Vertex, pos));
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offsetof(Vertex, nrm));
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offsetof(Vertex, tex));
```

[`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) isn't much different, the main thing with it is that it's one out of a two-parter with [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer).

In order to get out the same effect as the previous code we first need to make a call to [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) to quickly describe the data in the VBO. Despite *Bind* being in the name it isn't the same kind as for instance: `glBindTexture`.

Here's how they're both put into action:

```c
struct Vertex { vec3 pos, nrm; vec2 tex; };
 
glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glBindVertexBuffer(0, data->vbo, offsetof(Vertex, pos));
glBindVertexBuffer(1, data->vbo, offsetof(Vertex, nrm));
glBindVertexBuffer(2, data->vbo, offsetof(Vertex, tex));
    
glVertexAttribFormat(0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexAttribFormat(1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexAttribFormat(2, 2, GL_FLOAT, GL_FALSE, 0);
```

If you want to involve VAOs [`glEnableVertexArrayAttrib`](http://docs.gl/gl4/glEnableVertexAttribArray) comes into play, and [`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) & [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) transform into [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) & [`glVertexArrayAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat).

```c
glEnableVertexArrayAttrib(data->vao, 0);
glEnableVertexArrayAttrib(data->vao, 1);
glEnableVertexArrayAttrib(data->vao, 2);
    
glVertexArrayVertexBuffer(data->vao, 0, data->vbo, offsetof(Vertex, pos));
glVertexArrayVertexBuffer(data->vao, 1, data->vbo, offsetof(Vertex, nrm));
glVertexArrayVertexBuffer(data->vao, 2, data->vbo, offsetof(Vertex, tex));

glVertexArrayAttribFormat(data->vao, 0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 2, 2, GL_FLOAT, GL_FALSE, 0);
```

The version that takes in the VAO, [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer), has an equivalent for binding the IBO: [`glVertexArrayElementBuffer`](http://docs.gl/gl4/glVertexArrayElementBuffer).

All together this is how uploading an indexed model with *only* DSA should look:

```c
std::unique_ptr<Model> data(Model::Load("test.obj"));

glCreateBuffers(1, &data->vbo);	
glNamedBufferData(data->vbo, sizeof(Vertex)*data->vcount, data->vertices, GL_STATIC_DRAW);

glCreateBuffers(1, &data->ibo);
glNamedBufferData(data->ibo, sizeof(uint32_t)*data->icount, data->indices, GL_STATIC_DRAW);

glCreateVertexArrays(1, &data->vao);
	
glEnableVertexArrayAttrib(data->vao, 0);
glEnableVertexArrayAttrib(data->vao, 1);
glEnableVertexArrayAttrib(data->vao, 2);

glVertexArrayVertexBuffer(data->vao, 0, data->vbo, offsetof(Vertex, pos));
glVertexArrayVertexBuffer(data->vao, 1, data->vbo, offsetof(Vertex, nrm));
glVertexArrayVertexBuffer(data->vao, 2, data->vbo, offsetof(Vertex, tex));

glVertexArrayAttribFormat(data->vao, 0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 2, 2, GL_FLOAT, GL_FALSE, 0);

glVertexArrayElementBuffer(data->vao, data->ibo);
```

## Proper Way Of Retrieving All Uniform Names

Keep in mind the point of this little section is not to call anyone out but to present a more ideal way of going about things.

When I was getting my head around OpenGL I followed a particular YouTube tutor who covered OpenGL in respect to game development and did a pretty good job of it, however, there was one section in one of his videos which always irked me - and that was when he [demonstrated](https://github.com/BennyQBD/3DGameEngine/blob/master/src/com/base/engine/rendering/Shader.java#L176) his way of getting and storing all the names of his shader uniforms.

tl;dr: he parsed the shader sources himself! You don't have to do that!

Here is how you should do it:

```cpp
std::map<std::string, GLint> uniforms;
GLint uniform_count = 0;
GLsizei length = 0, size = 0;
GLenum type = GL_NONE;
glGetProgramiv(program_name, GL_ACTIVE_UNIFORMS, &uniform_count);

for (GLint i = 0; i < uniform_count; i++)
{
	std::array<GLchar, 0xff> uniform_name = {};
	glGetActiveUniform(program_name, i, uniform_name.size(), &length, &size, &type, uniform_name.data());
	uniforms[uniform_name.data()] = glGetUniformLocation(program_name, uniform_name.data());
}
```

With this you can do things like store the uniform datatype and check it in your uniform update functions.

Though really you should use UBOs instead of regular uniforms when you can.

## Alternative To Texture Atlases

The usage of texture atlases have been commonplace since the days of old and for good reason: less binds; it avoids the need to switch textures as fequently as you would otherwise. A popular example of its use is in Minecraft and are also almost always used for storing glyphs. 

Khronos recognises the advantages of atlases and devised a new set of texture types named GL_TEXTURE_1D_ARRAY, GL_TEXTURE_2D_ARRAY, and GL_TEXTURE_CUBE_MAP_ARRAY.

The point of the array is to give you all the advantages of using an atlas minus the inconvenience of having  to mess with texture coords.

To allocate a 2D texture array we do this:

```c
GLuint texarray = 0;
GLsizei width = 512, height = 512, layers = 3;
glCreateTextures(GL_TEXTURE_2D_ARRAY, 1, &texarray);
glTextureStorage3D(texarray, 0, GL_RGBA8, width, height, layers);
```

The lads over at Khronos decided to extend the use case of [`glTextureStorage3D`](http://docs.gl/gl4/glTexStorage3D) to be able to accommodate 2d texture arrays which I imagine is confusing at first but there's a pattern: the last dimension parameter acts as a layer specifier, so if you were to allocate a 1D texture array you would have to use the 2D storage function with height as the layer capacity.

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
	return max(0, min(capcity - 1, floor(layer + 0.5)));
}

void main()
{
	color = texture(texarray, vec3(tex0, layer2coord(3, diffuse_layer)));
}
```

Ideally you should calculate the layer coordinate outside of the shader.

You can take this way further and set up a little UBO/SSBO system of arrays containing layer id and texture array id pairs and update which layer id is used with regular uniforms.

Also, I advise against using ubos and ssbos for per object/draw stuff without a plan otherwise you will end up with everything not working as you'd like because the command queue has no involvement during the reads and writes.

## Faster Reads and Writes with Persistent Mapping

[`Persistent mapping`](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_buffer_storage.txt) was introduced in 4.3.
This will only work for immutable storage so if your buffers are never going to change size anyway I suggest moving to those even if you don't plan on using what I'm going to talk about. OpenGL will take it as a potential optimization hint.

With persistent mapping we can get a pointer to the memory our data is stored at and allow us to easily do very efficiently read and write to it even during drawing operations.

First we need the right flags for both buffer storage creation and the mapping itself:
```cpp
constexpr GLbitfield 
	mapping_flags = GL_MAP_WRITE_BIT | GL_MAP_READ_BIT | GL_MAP_PERSISTENT_BIT | GL_MAP_COHERENT_BIT,
	storage_flags = GL_DYNAMIC_STORAGE_BIT | mapping_flags;
```

* GL_MAP_COHERENT_BIT
	This flag ensures writes will be seen automagically by the server when done from the client and vice versa.
* GL_MAP_PERSISTENT_BIT
	This tells our driver you wish to hold onto the data despite what it's doing.
* GL_MAP_READ_BIT
	Lets OpenGL know we wish to read from the buffer so that it doesn't freak out when we do.
* GL_MAP_WRITE_BIT
	Lets OpenGL know we're gonna write to it, if you don't specify this *anything could happen*.

If we don't use these flags for the storage creation GL will reject your mapping request with scorn. What's worse is that you absolutely won't know unless you're doing some form of error checking.

Setting up our immutable storage is very basic:
```cpp
glCreateBuffers(1, &name);
glNamedBufferStorage(name, size, nullptr, storage_flags);
```
Whatever we put in the `const void* data` parameter is arbitrary and marking it as `nullptr` specifies we wish not to copy any data into it.

Here is how we get that pointer we were after:
```cpp
void* ptr = glMapNamedBufferRange(name, 0, size, mapping_flags);
```
You don't have to map it every frame and I advise against it as it harms overall performance.

Make sure to unmap the buffer before deleting it:
```cpp
glUnmapNamedBuffer(name);
glDeleteBuffers(1, &name);
```

Congratulations, you can now do `vertices[i] = Vertex{ ... };` whenever you like!
This means you can also use existing memory manipulation functions for copying, clearing, etc.

If you are going by the C++ standard I would strongly recommend storing the pointer somewhere difficult to derp on like a span:
```cpp
gsl::span<Vertex> vertices(reinterpret_cast<Vertex*>(glMapNamedBufferRange(name, 0, size, mapping_flags)), size);
```

`gsl::span<T>` is a non-owning container described in the [`C++ Core Guidelines`](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) github page and is available to use through Microsoft's implementation: the [`Guideline Support Library`](https://github.com/Microsoft/GSL).

Because it uses the standard STL container interface you can very easily use regular STL functions and ranged loops on it.

[Official wiki](https://www.khronos.org/opengl/wiki/Array_Texture).

## Informative articles

 * [DSA EXT specification](https://www.opengl.org/registry/specs/EXT/direct_state_access.txt).
 * [DSA ARB specification](https://www.opengl.org/registry/specs/ARB/direct_state_access.txt).
 * [Good-Reads-And-Tips-About-Programming](https://github.com/deccer/Good-Reads-And-Tips-About-Programming), all-around great resource.
 * Pretty much everything from the [OpenGL wiki](https://www.khronos.org/opengl/wiki/).
 
##### Have something you would like me to cover and/or fix? Let me know!
