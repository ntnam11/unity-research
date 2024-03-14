## Recipe - Combining Skinned Mesh Renderer

Today my request was to combining two different meshes.
I looked through Unity documentation, but there is no clear explanation on this topic.
(Well actually there are, but not enough for me to do what I need to...)

So, let's dig into Unity SkinnedMeshRenderer and find a recipe for combining them.

### Some basics

Let's make a simple comparison

||Mesh Renderer|Skinned Mesh Renderer|
|-|-|-|
|Component name|MeshRenderer|SkinnedMeshRenderer|
|Unity Documentation|[Mesh Renderer](https://docs.unity3d.com/Manual/class-MeshRenderer.html)|[Skinned Mesh Renderer](https://docs.unity3d.com/Manual/class-SkinnedMeshRenderer.html)
|Scripting API|[Mesh Renderer](https://docs.unity3d.com/ScriptReference/MeshRenderer.html)|[Skinned Mesh Renderer](https://docs.unity3d.com/ScriptReference/SkinnedMeshRenderer.html)
|Rendering|Mesh Filter|On its own|
|Rig-able|No|Yes|
|Animator|Not available|Using Avatar|
|Resources consumption|Low|High|

To access the mesh that is used to be skinned, we use `.sharedmesh`. This returns a `Mesh` object as `MeshFilter.sharedMesh` does. All vertex manipulations should be done on this object.

### Bones

#### Setting Bones

If your mesh was exported correctly with Armature (Blender) or Animation Controllers (3dsMax - idk if that's the correct term), you'll notice that there are empty GameObjects inside your model prefab (when you drag it into the Scene Hierarchy ofc).

These empty GameObjects form a bone structure, which will be referenced by our Skinned Mesh. Unity allows setting bones with [`SkinnedMeshRenderer.bones`](https://docs.unity3d.com/ScriptReference/SkinnedMeshRenderer-bones.html). In C# we'll do it like this:

```c#
Transform[] bones;
// getting transforms
// ...
// done getting transforms
skinnedMeshRenderer.bones = bones;
```

Our bone system will also need a root bone, where we can set it by using `SkinnedMeshRenderer.rootBone`. This will be our "root" of the skeleton.

### Baking Avatar

Well, you can't just animate a Skinned Mesh without an avatar, even if a correct bone structure was set up.

Here's how we gonna create it.

1. Create a bone structure - that is, a hierarchy of GameObjects
2. Call [`AvatarBuilder.BuildGenericAvatar`](https://docs.unity3d.com/ScriptReference/AvatarBuilder.BuildGenericAvatar.html) for a generic avatar, or [`AvatarBuilder.BuildHumanAvatar`](https://docs.unity3d.com/ScriptReference/AvatarBuilder.BuildHumanAvatar.html) if you want to create a Humanoid avatar. However please notice that the avatar itself must be valid*

\* Reference to [HumanDescription](https://docs.unity3d.com/ScriptReference/HumanDescription.html) for more information.

Example:

With this bone structure:

<img class="image" src="{{ site.baseurl }}/images/2024-03-14-sample-bone-structure.png" alt="Sample Bone Structure" />

and a little bit of code:
```c#
[MenuItem("GameObject/Create Avatar", false, 1000)]
public static void CreateAvatar()
{
    Avatar combinedAvatar = AvatarBuilder.BuildGenericAvatar(Selection.activeGameObject, string.Empty);
    combinedAvatar.name = "New Avatar";
    string path = EditorUtility.SaveFilePanelInProject("Select destination", "New Avatar", "asset", "Save new Avatar");
    if (!string.IsNullOrEmpty(path))
    {
        AssetDatabase.CreateAsset(combinedAvatar, path);
    }
}
```

we got this avatar ready for our skinned mesh combination:

<img class="image" src="{{ site.baseurl }}/images/2024-03-14-sample-bone-structure.png" alt="Sample New Avatar" />

#### Bindposes

Having only the bone system for our Skinned Mesh isn't enough. Our mesh needs `bindposes`, where we define the inverse transformation matrix of each bone. In simple terms, where the bone is located following the mesh.

Since the `bindposes` is defined for each bones, we need to set a `Matrix4x4[]`, with its length equals to the number of available bones. In other words:

```c#
Matrix4x4[] bindPoses = new Matrix4x4[bones.Length];
for (int i = 0; i < bones.Length; i++)
{
    bindPoses[i] = bones[i].worldToLocalMatrix * transform.localToWorldMatrix;
}
```

However setting bindposes is only needed if we don't already have bones and bindposes in our mesh (which, doesn't apply to our skinned mesh combiner).

We can get bindposes from the mesh by using [`mesh.GetBindposes`](https://docs.unity3d.com/ScriptReference/Mesh.GetBindposes.html)

#### Bone Weights

If you have worked with Blender or 3dsMax before, you'll know the concept of bone weight. In simple terms, bone weights define how much a vertex is affected by different bones.

Usually you'd like to use 4-bones skin weights (simple term: a vertex is affected by maximum of 4 bones). 1-bone and 2-bones may sometimes used, but it's better set up in Quality setting instead of the asset importer. Unity supports up to 255 bone weights per vertex (but I doubt if we even need that much).

This table summaries 2 methods can be used to handle bone weights.

|Struct used|[`BoneWeight`](https://docs.unity3d.com/ScriptReference/BoneWeight.html)|[`BoneWeight1`](https://docs.unity3d.com/ScriptReference/BoneWeight1.html)
|-|-|-|
|Description|Standard method of getting bone weights. Supports up to 4 bone weights per vertex.|Use this struct to support up to 255 bone weights per vertex. Also provide a little performance benefits|
|Getting bone weights|[`Mesh.boneWeights`](https://docs.unity3d.com/ScriptReference/Mesh-boneWeights.html)|[`Mesh.GetAllBoneWeights`](https://docs.unity3d.com/ScriptReference/Mesh.GetAllBoneWeights.html)|
|Return type|`BoneWeight[]`|`NativeArray<BoneWeight1>`|
|Non-allocation method|[`Mesh.GetBoneWeights`](https://docs.unity3d.com/ScriptReference/Mesh.GetBoneWeights.html)*|
|Setting bone weights|`Mesh.boneWeights`|[`Mesh.SetBoneWeights`](https://docs.unity3d.com/ScriptReference/Mesh.SetBoneWeights.html)|

\* *This method is used to avoid allocating a new array with every access. In simple terms, every time you use `Mesh.boneWeights`, Unity has to allocate a new array, therefore it's slower and consumes more memory.*

Example: What's inside bone weight struct:
```c#
Transform[] bones;

// by definition
BoneWeight[] boneWeights = new BoneWeight[vertexCount];
BoneWeight1[] boneWeight1s = new BoneWeight1[vertexCount];

// we have these for BoneWeight
BoneWeight boneWeight;
boneWeight.boneIndex0 = 4; // should be a number less than bones.Length
boneWeight.boneIndex1 = 2;
boneWeight.boneIndex2 = 0; // 0 means the root bone
boneWeight.boneIndex3 = 0;
boneWeight.weight0 = .05f; // a float defining weight
boneWeight.weight1 = .2f;
boneWeight.weight2 = 0;
boneWeight.weight3 = 0;

// meanwhile BoneWeight1 - equivalent to above BoneWeight
BoneWeight1 boneWeight10;
boneWeight10.boneIndex = 4;
boneWeight10.weight = .05f;
BoneWeight1 boneWeight11;
boneWeight11.boneIndex = 2;
boneWeight11.weight = .2f;
// no need to set 2 others since their weights are 0
```

### Combining Meshes

Unity provided a really convenient method for combining meshes: (`Mesh.CombineMeshes`)[https://docs.unity3d.com/ScriptReference/Mesh.CombineMeshes.html]

Usually we'll use it in combination with `CombineInstance[]`, in which we define which mesh to combine.

```c#
Mesh[] meshes;
// setting meshes
// ...
// end setting meshes

List<CombineInstance> combineInstances = new List<CombineInstance>();
for (int i = 0; i < meshes.Length; i++)
{
    combineInstances.Add(new CombineInstance
    {
        mesh = meshes[i],
        subMeshIndex = 0,
        transform = Matrix4x4.identity
    });
}
Mesh newMesh = new Mesh();
newMesh.CombineMeshes(combineInstances.ToArray(), true, true);
```

Notice the second parameter inside `CombineMeshes` function? It means we combine submeshes or not.

Unity stores materials following submeshes inside a mesh. In another words, each submesh has its own material ID. This could be a problem while combining meshes.

Depends on your needs, there are multiple ways to combine submeshes (for example, combine submesh index `0` of the first mesh with submesh index `1` of the second mesh, etc.). Material slots of combined mesh will be defined this way.

```c#
// getting submesh
SubMeshDescriptor subMeshDescriptor = mesh.GetSubMesh(i);

// setting submesh
mesh.SetSubMesh(subMeshIndex, subMeshDescriptor);
```

Reference to [SubMeshDescriptor Documentation](https://docs.unity3d.com/ScriptReference/Rendering.SubMeshDescriptor.html) for more information.

### BlendShapes 

If you're not familiar with BlendShapes , I recommend reading through the [Setting up Blendshapes in Unity](https://learn.unity.com/tutorial/setting-up-blendshapes-in-unity) tutorial. In short, BlendShapes are used to interpolate between sets of geometry, and mostly used for facial animations.

In Blender, blendshapes are Shape Keys, while in 3dsMax it's Morpher Modifier.

This is a template for getting/setting blendshapes

```c#
// Getting BlendShapes data
for (int i = 0; i < mesh.blendShapeCount; i++)
{
    Vector3[] deltaVertices = new Vector3[mesh.vertexCount];
    Vector3[] deltaNormals = new Vector3[mesh.vertexCount];
    Vector3[] deltaTangents = new Vector3[mesh.vertexCount];

    int shapeIndex = i;
    string shapeName = mesh.GetBlendShapeName(shapeIndex);

    int frameCount = mesh.GetBlendShapeFrameCount(shapeIndex);

    for (int frameIndex = 0; frameIndex < frameCount; frameIndex++)
    {
        mesh.GetBlendShapeFrameVertices(shapeIndex, frameIndex, deltaVertices, deltaNormals, deltaTangents);
        float frameWeight = mesh.GetBlendShapeFrameWeight(shapeIndex, frameIndex);

        // store these properties for further access:
        //
        // frameCount
        // shapeName
        // frameWeight
        // deltaVertices
        // deltaNormals
        // deltaTangents
        //
    }
}

// we'll need to loop through frameCount, then add frames using
mesh.AddBlendShapeFrame(shapeName, frameWeight, deltaVertices, deltaNormals, deltaTangents);
```

Used method and summary: 

|Method/Property|Params|Description|
|-|-|-|
|[`blendShapeCount`](https://docs.unity3d.com/ScriptReference/Mesh-blendShapeCount.html)||The number of BlendShape count of this mesh|
|[GetBlendShapeName](https://docs.unity3d.com/ScriptReference/Mesh.GetBlendShapeName.html)|`shapeIndex`|Returns name of BlendShape by its index|
|[GetBlendShapeFrameCount](https://docs.unity3d.com/ScriptReference/Mesh.GetBlendShapeFrameCount.html)|`shapeIndex`|Returns the frame count for BlendShape at index|
|[GetBlendShapeFrameVertices](https://docs.unity3d.com/ScriptReference/Mesh.GetBlendShapeFrameVertices.html)|`shapeIndex`<br>`frameIndex`<br>`deltaVertices`<br>`deltaNormals`<br>`deltaTangents`|Retreives deltaVertices, deltaNormals and deltaTangents of a blend shape frame. Each of these arrays has the same length as `mesh.vertexCount`|
|[GetBlendShapeFrameWeight](https://docs.unity3d.com/ScriptReference/Mesh.GetBlendShapeFrameWeight.html)|`shapeIndex`<br>`frameIndex`|Returns the weight of a blend shape frame|
|[AddBlendShapeFrame](https://docs.unity3d.com/ScriptReference/Mesh.AddBlendShapeFrame.html)|shapeName<br>frameWeight<br>`deltaVertices`<br>`deltaNormals`<br>`deltaTangents`|Add new Frame. Throw errors if the same name existed|

### Combining Skinned Mesh Renderer

Now that we've gathered enough ingredients for our combination. Let's sum up:

1. Creating bone structure (GameObject hierarchy)
2. Baking Avatar
3. Setting bindposes
4. Setting Bone Weights
5. Combining Meshes - dealing with Materials

I hope this recipe is enough for you to combine skinned meshes :\).