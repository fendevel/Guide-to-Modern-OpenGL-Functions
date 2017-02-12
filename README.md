# The Guide to Modern OpenGL Functions

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

###### glCreateTexture
* DSA equivalent of `glGenTextures`.

```c
void glCreateTextures(GLenum target, GLsizei n, GLuint *textures);
```

Unlike `glGenTextures` `glCreateTextures` will create the handle *and* initialize the object which is why the field `GLenum target`  is listed as the internal initialization depends on knowing the type.

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

* DSA equivalent of `glTexParameterX`

```c
void glTextureParameteri(GLuint texture, GLenum pname, GLenum param);
```

There isn't much to say about this family of functions; they're used exactly the same but take in the texture id rather than the texture target.

```c
glTextureParameteri(id, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
```

The `glTextureStorage` and `glTextureSubImage` families are the same exact way.

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

###### Generating Mip Maps

* DSA equivalent of `glGenerateMipmap`.

Takes in the texture instead of the texture target.

```c
void glGenerateTextureMipmap(GLuint texture);
```

###### Uploading Cube Maps

I should briefly point out that in order to upload cube map textures you need to use `glTextureImage2DEXT` as far as I know.

```c
for (size_t i = 0; i < 6; i++)
{
	const Bitmap& bitmap = bitmaps[i];
	glTextureImage2DEXT(id, GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 0, bitmap.internalFormat, bitmap.width,
    bitmap.height, 0, bitmap.format, GL_UNSIGNED_INT, bitmap.pixels);
}
```

### glFramebuffer
------
###### glCreateFramebuffers

* DSA equivalent of `glGenFramebuffers`.

```c
void glGenFramebuffers(GLsizei n, GLuint* framebuffers);
```
Used exactly like `glGenFramebuffers` but initializes the object for you.

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
* DSA equivalent of `glGenBuffers`.

```c
void glCreateBuffers(GLsizei n, GLuint* buffers);
```

Used exactly like its traditional equivalent and automically initializes the object.

###### glNamedBufferData
* DSA equivalent of `glBufferData`

```c
void glNamedBufferData(GLuint buffer, GLsizei size, const GLvoid *data, GLenum usage);
```

Same exact use case as `glBufferData` but instead of requiring the buffer target it takes in the buffer handle itself.

###### glVertexAttribFormat
* DSA equivalent of `glVertexAttribPointer`.

```c
void glVertexAttribFormat(GLuint attribindex, GLint size, GLenum type, GLboolean normalized, GLuint relativeoffset);
```

If you aren't familiar with the application of `glVertexAttribPointer` it is called like so:

* ***GLuint index***: location of the shader attribute for it to get mapped to. 
* ***GLint size***: number of components that make up the attribute, ranges from 1 to 4.
* ***GLenum type***: the type each component is stored as.
* ***GLboolean normalized***: specifies whether the attribute should be normalized.
* ***GLsizei stride***: size of each element that makes up the array buffer (in bytes), so a vec3 would be `sizeof(float)*3`.
* ***const GLvoid(ptr) pointer***: where the attribute begins in he buffer's memory, so a vec4 going after a vec3 would use an offset of `(void*)(sizeof(float)*3)` while the vec would be at zero.

```c
struct Vertex { vec3 pos, nrm; vec2 tex; };
unsigned int offset = 0u;

glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offset));											
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offset += sizeof(vec3)));
glVertexAttribPointer(2, 2, GL_FLOAT, GL_FALSE, sizeof(Vertex), (void*)(offset += sizeof(vec3)));
```

`glVertexAttribFormat` isn't much different:

* ***GLuint attribindex***: location of the shader attribute for it to get mapped to.
* ***GLint size***: number of components that make up the attribute, ranges from 1 to 4.
* ***GLenum type***: the type each component is stored as.
* ***GLboolean normalized***: specifies whether the attribute should be normalized.
* ***GLuint relativeoffset***: the distance between this attribute and the last.

In order to get out the same effect as the previous code we first need to make a call to `glBindVertexBuffer` to lightly describe the data in the VBO. Despite *Bind* being in the function symbol I'm pretty sure it isn't the same kind as for instance: `glBindTexture`.

###### glBindVertexBuffer

Here is how `glBindVertexBuffer` is layed out:

* ***GLuint bindingindex***: index of the attribute
* ***GLuint buffer***: the buffer, like a vbo
* ***GLintptr offset***: the absolute offset of the attribute data in the buffer, the same as the `const GLvoid* pointer` parameter but no need to cast to a void pointer.
* ***GLintptr stride***: size of each element that makes up the array buffer.

```c
struct Vertex { vec3 pos, nrm; vec2 tex; };
unsigned int offset = 0u;
    
glEnableVertexAttribArray(0);
glEnableVertexAttribArray(1);
glEnableVertexAttribArray(2);

glBindVertexBuffer(0, vbo, offset, sizeof(Vertex));
glBindVertexBuffer(1, vbo, offset += sizeof(vec3), sizeof(Vertex));
glBindVertexBuffer(2, vbo, offset += sizeof(vec3), sizeof(Vertex));
    
glVertexAttribFormat(0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexAttribFormat(1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexAttribFormat(2, 2, GL_FLOAT, GL_FALSE, 0);
```

If you want to involve VAOs `glEnableVertexArrayAttrib`, `glVertexArrayVertexBuffer` and `glVertexArrayAttribFormat` come into play.

```c
glEnableVertexArrayAttrib(vao, 0);
glEnableVertexArrayAttrib(vao, 1);
glEnableVertexArrayAttrib(vao, 2);
    
glVertexArrayVertexBuffer(vao, 0, vbo, offset, sizeof(Vertex));
glVertexArrayVertexBuffer(vao, 1, vbo, offset += sizeof(vec3), sizeof(Vertex));
glVertexArrayVertexBuffer(vao, 2, vbo, offset += sizeof(vec3), sizeof(Vertex));

glVertexArrayAttribFormat(vao, 0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(vao, 1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(vao, 2, 2, GL_FLOAT, GL_FALSE, 0);
```

The version that takes in the VAO, `glVertexArrayVertexBuffer`, has an equivalent for binding the IBO 
```c
void glVertexArrayElementBuffer(GLuint vaobj, GLuint buffer);
```

All together this is how uploading an indexed model with *only* DSA functions should look:

```c
Model* data = Model::Load("test.obj");
size_t offset = 0;
glCreateBuffers(1, &data->vbo);	
glNamedBufferData(data->vbo, sizeof(Vertex)*data->vcount, data->vertices, GL_STATIC_DRAW);

glCreateBuffers(1, &data->ibo);
glNamedBufferData(data->ibo, sizeof(unsigned int)*data->icount, data->indices, GL_STATIC_DRAW);

glCreateVertexArrays(1, &data->vao);
	
glEnableVertexArrayAttrib(data->vao, 0);
glEnableVertexArrayAttrib(data->vao, 1);
glEnableVertexArrayAttrib(data->vao, 2);

glVertexArrayVertexBuffer(data->vao, 0, data->vbo, offset, sizeof(Vertex));
glVertexArrayVertexBuffer(data->vao, 1, data->vbo, offset += sizeof(float3), sizeof(Vertex));
glVertexArrayVertexBuffer(data->vao, 2, data->vbo, offset += sizeof(float3), sizeof(Vertex));

glVertexArrayAttribFormat(data->vao, 0, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 1, 3, GL_FLOAT, GL_FALSE, 0);
glVertexArrayAttribFormat(data->vao, 2, 2, GL_FLOAT, GL_FALSE, 0);
glVertexArrayElementBuffer(data->vao, data->ibo);
```

## Alternative To Texture Atlases

The usage of texture atlases have been commonplace since the days of old and for good reason: less binds; it avoids the need to switch texture as fequently as you would otherwise. A popular example of its use is in Minecraft and are also almost always used for storing glyphs. 

Khronos recognises the advantages of atlases and devised a new set of texture types named GL_TEXTURE_1D_ARRAY, GL_TEXTURE_2D_ARRAY, and GL_TEXTURE_CUBE_MAP_ARRAY.

The point of the array is to give you all the advantages of using an atlas minus the inconvenience of having  to mess with texture coords.

To allocate a 2D texture array we do this:

```c
	GLuint texarray = 0;
	GLsizei width = 512, height = 512, layers = 3;
	glCreateTextures(GL_TEXTURE_2D_ARRAY, 1, &texarray);
	glTextureStorage3D(texarray, 0, GL_RGBA8, width, height, layers);
```

The lads over at Khronos decided to extend the purpose of `glTextureStorage3D` to be able to accomidate 2d texture arrays which I imagine is confusing at first but there's a pattern: the last dimension parameter acts as a layer specifier, so if you were to allocate a 1D texture array you would have to use the 2D storage function and use height as the layer capacity.

Anyway, uploading to individual layers is very straght forward:

```c
	glTextureSubImage3D(texarray, mipmap_level, offset.x, offset.y, layer, width, height, 1, GL_RGBA, GL_UNSIGNED_BYTE, pixels);
```

It's super duper simple.

The largest different between arrays and atlases in terms of implementation lies in the shader.

To bind a texture array to the context you need a specialized sampler called `sampler2DArray`. We will also need a uniform to store the layer id.

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

You can take this way further and set up a little system of arrays containing layer id and texture array id pairs and update it through a UBO or SSBO.

[Official wiki](https://www.khronos.org/opengl/wiki/Array_Texture).

## Informative articles

 * [DSA EXT specification](https://www.opengl.org/registry/specs/EXT/direct_state_access.txt).
 * [DSA ARB specification](https://www.opengl.org/registry/specs/ARB/direct_state_access.txt).
 * [Good-Reads-And-Tips-About-Programming](https://github.com/deccer/Good-Reads-And-Tips-About-Programming), all-around great resource for programmers.
 * Pretty much everything from the [OpenGL wiki](https://www.khronos.org/opengl/wiki/).
 
##### Have something you would like me to cover? Let me know!