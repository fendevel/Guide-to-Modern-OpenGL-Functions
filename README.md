# A Guide to Modern OpenGL Functions

## Index

[DSA](#dsa-direct-state-access)

[glTexture](#gltexture)

[glFramebuffer](#glframebuffer)

[glBuffer](#glbuffer)

[Proper Way Of Retrieving All Uniform Names](#proper-way-of-retrieving-all-uniform-names)

[Solution To Texture Atlases](#solution-to-texture-atlases)

[Texture Views & Aliases](#texture-views--aliases)

[Setting up Mix & Match Shaders with Program Pipelines](#setting-up-mix--match-shaders-with-program-pipelines)

[Faster Reads and Writes with Persistent Mapping](#faster-reads-and-writes-with-persistent-mapping)

[More Information](#more-information)

What this is:

 * A guide on how to properly apply scarcely documented modern OpenGL functions.

What this is not:

* A guide on modern OpenGL rendering techniques.

When I say modern I'm talking DSA modern, not VAO modern, because that's old modern or "middle" gl (however I will be covering some stuff from around that version), I can't tell you what minimal version you need to make use of DSA because it's not clear at all but you can check if you support it yourself with something like glew's `glewIsSupported("ARB_direct_state_access")`.

## DSA (Direct State Access)
With DSA we, in theory, can keep our bind count outside of drawing operations at zero, great right? Sure, but if you were to research how to use all the new DSA functions you'd have a hard time finding anywhere where it's all explained, which is what this guide is all about.

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
* The texture related calls aren't complex to figure out so let's jump right in.

###### glCreateTextures
* [`glCreateTextures`](http://docs.gl/gl4/glCreateTextures) is the equivalent of [`glGenTextures`](http://docs.gl/gl4/glGenTextures) + [`glBindTexture`](http://docs.gl/gl4/glBindTexture)(for initialization).

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

glTexStorage2D(GL_TEXTURE_2D, 1, GL_RGBA8, width, height);
glTexSubImage2D(GL_TEXTURE_2D, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_INT, pixels);
```

```c
glCreateTextures(GL_TEXTURE_2D, 1, &id);

glTextureParameteri(id, GL_TEXTURE_WRAP_S, GL_CLAMP);
glTextureParameteri(id, GL_TEXTURE_WRAP_T, GL_CLAMP);
glTextureParameteri(id, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTextureParameteri(id, GL_TEXTURE_MAG_FILTER, GL_NEAREST);

glTextureStorage2D(id, 1, GL_RGBA8, width, height);
glTextureSubImage2D(id, 0, 0, 0, width, height, GL_RGBA, GL_UNSIGNED_INT, pixels);
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
struct vertex_t { vec3 pos, nrm; vec2 tex; };

glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, pos));
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, nrm));
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(vertex_t), (void*)(offsetof(vertex_t, tex));
```

[`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat) isn't much different, the main thing with it is that it's one out of a two-parter with [`glVertexAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding).

In order to get out the same effect as the previous code we first need to make a call to [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer). Despite *Bind* being in the name it isn't the same kind as for instance: `glBindTexture`.

Here's how they're both put into action:

```c
struct vertex_t { vec3 pos, nrm; vec2 tex; };

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

If you want to involve VAOs then [`glEnableVertexAttribArray`](http://docs.gl/gl4/glEnableVertexAttribArray), [`glVertexAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat), [`glVertexAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding), and [`glBindVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer) transform into [`glEnableVertexArrayAttrib`](http://docs.gl/gl4/glEnableVertexAttribArray), [`glVertexArrayAttribFormat`](http://docs.gl/gl4/glVertexAttribFormat), [`glVertexArrayAttribBinding`](http://docs.gl/gl4/glVertexAttribBinding), and [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer).

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

The version that takes in the VAO, [`glVertexArrayVertexBuffer`](http://docs.gl/gl4/glBindVertexBuffer), has an equivalent for binding the IBO: [`glVertexArrayElementBuffer`](http://docs.gl/gl4/glVertexArrayElementBuffer).

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

## Proper Way Of Retrieving All Uniform Names

Keep in mind the point of this little section is not to call anyone out but to present a more ideal way of going about things.

When I was getting my head around OpenGL I followed a particular YouTube tutor who covered OpenGL in respect to game development and did a pretty good job of it, however, there was one section in one of his videos which always irked me - and that was when he [demonstrated](https://github.com/BennyQBD/3DGameEngine/blob/master/src/com/base/engine/rendering/Shader.java#L176) his way of getting and storing all the names of his shader uniforms.

tl;dr: he parsed the shader sources himself! You don't have to do that!

Here is how you should do it:

```cpp
struct uniform_info_t
{ 
	GLint location;
	GLsizei count;
};

std::map<std::string, uniform_info_t> uniforms;
GLint uniform_count = 0;
GLsizei length = 0, size = 0;
GLenum type = GL_NONE;
glGetProgramiv(program_name, GL_ACTIVE_UNIFORMS, &uniform_count);

for (GLint i = 0; i < uniform_count; i++)
{
	std::array<GLchar, 0xff> uniform_name = {};
	glGetActiveUniform(program_name, i, uniform_name.size(), &length, &size, &type, uniform_name.data());
	
	uniform_info_t uniform_info = {};
	uniform_info.location = glGetUniformLocation(program_name, uniform_name.data());
	uniform_info.count = size;
	
	uniforms.emplace(std::make_pair(std::string(uniform_name.data(), length), uniform_info));
}
```

Note that the `GLsizei size` parameter refers to the number of locations the uniform takes up with `mat3`, `vec4`, `float`, etc being 1 and any array having the same size as its element count, this means that if you want to modify element 5 of uniform `my_array` you will need to do:
```cpp
glProgramUniformXX(program_name, uniforms["my_array[0]"].location + 5, value);
``` 
or if you want to modify the whole array:
```cpp
glProgramUniformXXv(program_name, uniforms["my_array[0]"].location, uniforms["my_array[0]"].count, my_array);
```

With this you can do things like store the uniform datatype and check it in your uniform update functions.

Though really you should use UBOs instead of regular uniforms when you can.

## Solution To Texture Atlases

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

They decided to extend the use case of [`glTextureStorage3D`](http://docs.gl/gl4/glTexStorage3D) to be able to accommodate 2d texture arrays which I imagine is confusing at first but there's a pattern: the last dimension parameter acts as a layer specifier, so if you were to allocate a 1D texture array you would have to use the 2D storage function with height as the layer capacity.

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

As a bonus let me tell you an easy way to populate a texture array with parts of an atlas, in our case an rpg tileset.

Modern OpenGL comes with two generic raw memory copying functions: [`glCopyImageSubData`](http://docs.gl/gl4/glCopyImageSubData) and [`glCopyBufferSubData`](http://docs.gl/gl4/glCopyBufferSubData). Here we'll be dealing with [`glCopyImageSubData`](http://docs.gl/gl4/glCopyImageSubData), this function allows us to copy sections of a source image to a region of a destination image.
We're going to take advantage of its offset and size parameters so that we can copy tiles from every location and paste them in the appropriate layers within our texture array.

Here it is:
```cpp
GLsizei image_w, image_h, c, tile_w = 16, tile_h = 16;
stbi_uc* pixels = stbi_load(".\\textures\\tiles_packed.png", &image_w, &image_h, &c, 4);
GLuint tileset;
GLsizei
	tiles_x = (image_w / tile_w),
	tiles_y = (image_h / tile_h),
	tile_count = (image_w / tile_w)*(image_h / tile_h);

glCreateTextures(GL_TEXTURE_2D_ARRAY, 1, &tileset);
glTextureStorage3D(tileset, 1, GL_RGBA8, tile_w, tile_h, tile_count);

{
	GLuint temp_tex = 0;
	glCreateTextures(GL_TEXTURE_2D, 1, &temp_tex);
	glTextureStorage2D(temp_tex, 1, GL_RGBA8, image_w, image_h);
	glTextureSubImage2D(temp_tex, 0, 0, 0, image_w, image_h, GL_RGBA, GL_UNSIGNED_BYTE, pixels);

	for (GLsizei i = 0; i < tile_count; i++)
	{
		GLint x = (i % tiles_x), y = (i / tiles_x);
		glCopyImageSubData(temp_tex, GL_TEXTURE_2D, 0, x, y, 0, tileset, GL_TEXTURE_2D_ARRAY, 0, 0, 0, i, tile_w, tile_h, 1);
	}
	glDeleteTextures(1, &temp_tex);
}

stbi_image_free(pixels);
```

## Texture Views & Aliases

Texture views allow us to share a section of a texture's storage with an object of a different texture target. *Share* as in there's no copying at all, any changes you make to a view is visible to all views that share the storage along with the original texture. This is useful because we can read and write data to these sections as if they were of the target and format the view interprets it as, we are also able to bind and use them as regular textures.

The original storage is only freed once all references to it are deleted, if you are familiar with C++'s `std::shared_ptr` it's very similar. 

We can only make views if we use a target and format which is compatible with our original texture, here are some tables from the [wiki](https://www.khronos.org/opengl/wiki/Texture_Storage#View_texture_aliases):


| Original Targets|Compatible New Targets|
|:----------------|:---------------------|
| GL_TEXTURE_1D	|GL_TEXTURE_1D, GL_TEXTURE_1D_ARRAY|
| GL_TEXTURE_2D |GL_TEXTURE_2D, GL_TEXTURE_2D_ARRAY|
| GL_TEXTURE_3D	|GL_TEXTURE_3D|
| GL_TEXTURE_CUBE_MAP |GL_TEXTURE_CUBE_MAP, GL_TEXTURE_2D, GL_TEXTURE_2D_ARRAY, GL_TEXTUER_CUBE_MAP_ARRAY|
| GL_TEXTURE_RECTANGLE |GL_TEXTURE_RECTANGLE|
| GL_TEXTURE_BUFFER |none. Cannot be used with this function.|
| GL_TEXTURE_1D_ARRAY |GL_TEXTURE_1D, GL_TEXTURE_1D_ARRAY|
| GL_TEXTURE_2D_ARRAY |GL_TEXTURE_2D, GL_TEXTURE_CUBE_MAP, GL_TEXTURE_2D_ARRAY, GL_TEXTUER_CUBE_MAP_ARRAY|
| GL_TEXTURE_CUBE_MAP_ARRAY | GL_TEXTURE_CUBE_MAP, GL_TEXTURE_2D, GL_TEXTURE_2D_ARRAY, GL_TEXTUER_CUBE_MAP_ARRAY|
| GL_TEXTURE_2D_MULTISAMPLE |GL_TEXTURE_2D_MULTISAMPLE, GL_TEXTURE_2D_MULTISAMPLE_ARRAY|
| GL_TEXTURE_2D_MULTISAMPLE_ARRAY |GL_TEXTURE_2D_MULTISAMPLE, GL_TEXTURE_2D_MULTISAMPLE_ARRAY|

| Class	| Internal Formats |
|:------|:-----------------|
| 128-bit |	GL_RGBA32F, GL_RGBA32UI, GL_RGBA32I |
| 96-bit |GL_RGB32F, GL_RGB32UI, GL_RGB32I |
| 64-bit |GL_RGBA16F, GL_RG32F, GL_RGBA16UI, GL_RG32UI, GL_RGBA16I, GL_RG32I, GL_RGBA16, GL_RGBA16_SNORM |
| 48-bit |GL_RGB16, GL_RGB16_SNORM, GL_RGB16F, GL_RGB16UI, GL_RGB16I |
| 32-bit |GL_RG16F, GL_R11F_G11F_B10F, GL_R32F, GL_RGB10_A2UI, GL_RGBA8UI, GL_RG16UI, GL_R32UI, GL_RGBA8I, GL_RG16I, GL_R32I, GL_RGB10_A2, GL_RGBA8, GL_RG16, GL_RGBA8_SNORM, GL_RG16_SNORM, GL_SRGB8_ALPHA8, GL_RGB9_E5 |
| 24-bit |GL_RGB8, GL_RGB8_SNORM, GL_SRGB8, GL_RGB8UI, GL_RGB8I |
| 16-bit |GL_R16F, GL_RG8UI, GL_R16UI, GL_RG8I, GL_R16I, GL_RG8, GL_R16, GL_RG8_SNORM, GL_R16_SNORM |
| 8-bit	|GL_R8UI, GL_R8I, GL_R8, GL_R8_SNORM |
| GL_VIEW_CLASS_RGTC1_RED	|GL_COMPRESSED_RED_RGTC1, GL_COMPRESSED_SIGNED_RED_RGTC1 |
| GL_VIEW_CLASS_RGTC2_RG	|GL_COMPRESSED_RG_RGTC2, GL_COMPRESSED_SIGNED_RG_RGTC2 |
| GL_VIEW_CLASS_BPTC_UNORM	|GL_COMPRESSED_RGBA_BPTC_UNORM, GL_COMPRESSED_SRGB_ALPHA_BPTC_UNORM |
| GL_VIEW_CLASS_BPTC_FLOAT	|GL_COMPRESSED_RGB_BPTC_SIGNED_FLOAT, GL_COMPRESSED_RGB_BPTC_UNSIGNED_FLOAT |

S3TC texture view compatibility

|Class | Internal formats |
|:-----|:-----------------|
| GL_S3TC_DXT1_RGB	|GL_COMPRESSED_RGB_S3TC_DXT1_EXT, GL_COMPRESSED_SRGB_S3TC_DXT1_EXT		 |
| GL_S3TC_DXT1_RGBA	|GL_COMPRESSED_RGBA_S3TC_DXT1_EXT, GL_COMPRESSED_SRGB_ALPHA_S3TC_DXT1_EXT|
| GL_S3TC_DXT3_RGBA	|GL_COMPRESSED_RGBA_S3TC_DXT3_EXT, GL_COMPRESSED_SRGB_ALPHA_S3TC_DXT3_EXT|
| GL_S3TC_DXT5_RGBA	|GL_COMPRESSED_RGBA_S3TC_DXT5_EXT, GL_COMPRESSED_SRGB_ALPHA_S3TC_DXT5_EXT|

Making the view itself is extremely simple, it's only two function calls: [`glGenTextures`](http://docs.gl/gl4/glGenTextures) & [`glTextureView`](http://docs.gl/gl4/glTextureView).

Despite this being about modern OpenGL [`glGenTextures`](http://docs.gl/gl4/glGenTextures) has an important role here: we need only an available texture name and nothing more; this needs to be a completely empty uninitialized object and since this function only handles the generation of a valid handle it's perfect for this.

[`glTextureView`](http://docs.gl/gl4/glTextureView) is what will do the main view creation.

If we were to have a typical 2d texture array and needed a view of layer 5 in isolation this is how it would look:

```c
glGenTextures(1, &view_name);
glTextureView(view_name, GL_TEXTURE_2D, texarray_name, GL_RGBA8, min_level, level_count, 5, 1);
```

With this you can bind layer 5 alone as a `GL_TEXTURE_2D` texture.

This is the exact same when dealing with cube maps, the layer parameters will correspond to the cube faces with the layer params of cube map arrays being `cubemap_layer * 6 + face_layer`.

Outside the storage sharing property views are pretty much just regular texture objects, this means we can make views of other views and have some fun viewseption magic, we could have views from slices of a larger view array.

The fact that we can specify which mipmaps we want in the view means that we can have views which are just of those specific mipmap levels, so for example you could make textures views of the *N*th mipmap level of a bunch of textures and use only those for expensive texture dependant lighting calculations.

## Setting up Mix & Match Shaders with Program Pipelines

[Program Pipeline](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_separate_shader_objects.txt) objects allow us to change shader stages on the fly without having to relink them, this is what most if not all hardware, including game consoles, are targetted towards. OpenGL's default way of tightly packing stages together into a single program allows for better optimization but often it's not worth it when conisdering the benefits of the mix-and-match approach.

To create and set up a simple program pipeline without any debugging looks like this:

```cpp
std::array<const char*, 1>
	vs_source = { load_file(".\\main_shader.vs").c_str() },
	fs_source = { load_file(".\\main_shader.fs").c_str() };
	
GLuint 
	vs = glCreateShaderProgramv(GL_VERTEX_SHADER, 1, vs_source.data()),
	fs = glCreateShaderProgramv(GL_FRAGMENT_SHADER, 1, fs_source.data()),
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
 in gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
} gl_in[gl_MaxPatchVertices];

out gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
} gl_out[];
```

Tesselation Evaluation:
```glsl
in gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
} gl_in[gl_MaxPatchVertices];

out gl_PerVertex {
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
};
```

Geometry:
```glsl
in gl_PerVertex
{
  vec4 gl_Position;
  float gl_PointSize;
  float gl_ClipDistance[];
} gl_in[];

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

**note: the speed afforded by this features greatly varies between vendors**

[`Persistent mapping`](https://www.khronos.org/registry/OpenGL/extensions/ARB/ARB_buffer_storage.txt) was introduced in 4.3.
This will only work for immutable storage so if your buffers are never going to change size anyway I suggest moving to those even if you don't plan on using what I'm going to talk about. OpenGL will take it as a potential optimization hint.

With persistent mapping we can get a pointer to the memory our data is stored at and allow us to easily and very efficiently read and write to it even during drawing operations.

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

Setting up our immutable storage is very basic:
```cpp
glCreateBuffers(1, &name);
glNamedBufferStorage(name, size, nullptr, storage_flags);
```
Whatever we put in the `const void* data` parameter is arbitrary and marking it as `nullptr` specifies we wish not to copy any data into it.

Here is how we get that pointer we were after:
```cpp
void* ptr = glMapNamedBufferRange(name, offset, size, mapping_flags);
```
You don't have to map it every frame and I advise against it as it harms overall performance.

Make sure to unmap the buffer before deleting it:
```cpp
glUnmapNamedBuffer(name);
glDeleteBuffers(1, &name);
```

Congratulations, you can now do `vertices[i] = vertex_t{ ... };` whenever you like!
This means you can also use existing memory manipulation functions for copying, clearing, etc.

If you are going by the C++ standard I would strongly recommend storing the pointer somewhere difficult to derp on like a span:
```cpp
gsl::span<vertex_t> vertices(reinterpret_cast<vertex_t*>(glMapNamedBufferRange(name, 0, size, mapping_flags)), size);
```

`gsl::span<T>` is a non-owning container described in the [`C++ Core Guidelines`](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) github page and is available to use through Microsoft's implementation: the [`Guideline Support Library`](https://github.com/Microsoft/GSL).

Because it uses the standard STL container interface you can very easily use regular STL functions and ranged loops on it.

[Official wiki](https://www.khronos.org/opengl/wiki/Array_Texture).

## More information

 * [DSA EXT specification](https://www.opengl.org/registry/specs/EXT/direct_state_access.txt).
 * [DSA ARB specification](https://www.opengl.org/registry/specs/ARB/direct_state_access.txt).
 * [Good-Reads-And-Tips-About-Programming](https://github.com/deccer/Good-Reads-And-Tips-About-Programming), all-around great resource.
 * Pretty much everything from the [OpenGL wiki](https://www.khronos.org/opengl/wiki/).
 
##### Have something you would like me to cover and/or fix? Let me know!
