# OpenGL 3D Character Rendering System

OpenGL 3.3으로 마인크래프트 스타일의 3D 캐릭터 렌더링 시스템을 처음부터 구현했습니다. 페이퍼크래프트 이미지에서 UV 좌표를 수작업으로 추출하여 텍스처 매핑을 적용하고, 3개의 서로 다른 애니메이션 패턴을 가진 캐릭터를 제작했습니다. Virtual Trackball 카메라 시스템과 Environment Mapping(Skybox + 굴절 효과)도 직접 구현했습니다. 
(전체 코드는 저작권 상 올리지 못해 직접 구현한 부분만 적습니다.)


---

### 1. Manual UV Mapping

#### 1.1 Texture Coordinate Extraction

페이퍼크래프트 이미지를 분석하여 각 큐브 면의 모서리 픽셀 좌표를 찾고, 이를 [0,1] 범위의 UV 좌표로 수동 변환했습니다. 캐릭터의 머리, 몸통, 팔, 다리 부위마다 각각 48개씩(6면 x 4정점 x 2좌표) 총 240개의 UV 좌표를 직접 계산했습니다.

예시: 머리 부위의 UV 좌표 배열

```cpp
float texHead[] = {
    // front face (4 vertices x 2 coordinates)
    0.550, 0.503,  0.755, 0.503,  0.755, 0.685,  0.550, 0.685,
    // back face
    0.345, 0.503,  0.345, 0.685,  0.141, 0.685,  0.141, 0.503,
    // left, right, top, bottom faces...
};

float texBody[] = {
    // 6 faces x 4 vertices x 2 coordinates
    0.304, 0.214,  0.174, 0.214,  0.174, 0.048,  0.304, 0.048,
    // ...
};

// texLeftArm[], texRightArm[], texLeg[] arrays similarly defined
```

#### 1.2 Multi-Texture System

5개의 2D 텍스처를 동적으로 바인딩하는 시스템을 구현했습니다. 3개의 캐릭터 스킨과 2개의 환경 텍스처(grass, stone)를 각각 독립된 GL_TEXTURE_2D 객체로 관리하며, 렌더링 시 필요한 텍스처로 실시간 전환됩니다.

```cpp
GLuint color_tex[5];

for (int i = 0; i < 5; i++) {
    LoadBMPFile(&dst_ch[i], &width_ch[i], &height_ch[i], filename[i]);
    
    glGenTextures(1, &color_tex[i]);
    glBindTexture(GL_TEXTURE_2D, color_tex[i]);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
    glTexImage2D(GL_TEXTURE_2D, 0, GL_RGBA, width_ch[i], height_ch[i], 
                 0, GL_RGBA, GL_UNSIGNED_BYTE, dst_ch[i]);
}
```

---

### 2. Full-Body Character System

#### 2.1 Hierarchical Body Structure

큐브 프리미티브 5개(머리, 몸통, 좌우 팔, 다리)를 변환 행렬로 조합하여 완전한 인간형 캐릭터를 구현했습니다. 각 신체 부위의 크기와 위치를 수작업으로 조정하여 자연스러운 비율을 만들었습니다.

```cpp
void drawCharacter(mat4 baseTransform, mat4 view, mat4 proj, int idx)
{
    mat4 animatedTransform = baseTransform;
    
    // Animation transformations applied here
    
    // Head: 0.6x0.6x0.6 scale, position at Y=1.5
    mat4 modelHead = animatedTransform * Translate(0.0f, 1.5f, 0.0f) 
                                       * Scale(0.6f, 0.6f, 0.6f);
    if (idx == 0) modelHead = modelHead * Rotate(animHeadRot, vec3(0, 1, 0));
    drawCube(VAO_head[idx], modelHead, view, proj);
    
    // Body: 0.6x0.8x0.3 scale, position at Y=0.8
    mat4 modelBody = animatedTransform * Translate(0.0f, 0.8f, 0.0f) 
                                       * Scale(0.6f, 0.8f, 0.3f);
    drawCube(VAO_body[idx], modelBody, view, proj);
    
    // Left Arm: 0.25x0.9x0.3 scale, position at X=-0.425, Y=0.75
    mat4 modelLeftArm = animatedTransform * Translate(-0.425f, 0.75f, 0.0f) 
                                          * Scale(0.25f, 0.9f, 0.3f);
    drawCube(VAO_leftArm[idx], modelLeftArm, view, proj);
    
    // Right Arm: 0.25x0.9x0.3 scale, position at X=0.425, Y=0.75
    mat4 modelRightArm = animatedTransform * Translate(0.425f, 0.75f, 0.0f) 
                                           * Scale(0.25f, 0.9f, 0.3f);
    drawCube(VAO_rightArm[idx], modelRightArm, view, proj);
    
    // Legs: 0.6x0.8x0.25 scale, origin position
    mat4 modelLeg = animatedTransform * Translate(0.0f, 0.0f, 0.0f) 
                                      * Scale(0.6f, 0.8f, 0.25f);
    drawCube(VAO_leg[0], modelLeg, view, proj);
}
```

#### 2.2 VAO/VBO Architecture Design

메모리 효율을 위해 정점 위치 데이터는 공유 VBO로, 텍스처 좌표는 캐릭터별 독립 VBO로 분리하는 구조를 설계했습니다. 각 캐릭터마다 5개 신체 부위 x 3명 = 총 15개의 VAO를 생성하여 렌더링 시 빠른 상태 전환이 가능하도록 했습니다.

```cpp
for (int i = 0; i < 3; i++) {
    // Head VAO setup
    glGenVertexArrays(1, &VAO_head[i]);
    glBindVertexArray(VAO_head[i]);
    
    // Position attribute (location=0, shared VBO)
    glBindBuffer(GL_ARRAY_BUFFER, VBO_pos);
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3*sizeof(float), 0);
    glEnableVertexAttribArray(0);
    
    // Texture coordinate attribute (location=1, per-character VBO)
    glGenBuffers(1, &VBO_head[i]);
    glBindBuffer(GL_ARRAY_BUFFER, VBO_head[i]);
    glBufferData(GL_ARRAY_BUFFER, sizeof(texHead), texHead, GL_STATIC_DRAW);
    glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 2*sizeof(float), 0);
    glEnableVertexAttribArray(1);
    
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, IBO);
    
    // Body, Arms, Legs similarly configured...
}
```

---

### 3. Animation System (No Keyframes)

키프레임 없이 수학적 계산만으로 3가지 서로 다른 애니메이션 패턴을 구현했습니다. idle() 콜백에서 매 프레임 애니메이션 상태를 업데이트하고, 이를 변환 행렬에 적용하는 방식입니다.

#### 3.1 Character 1: Continuous Y-Axis Rotation

머리가 Y축 기준으로 계속 회전하는 애니메이션을 구현했습니다. 각도를 누적하며 360도 초과 시 자동 리셋되도록 처리했습니다.

```cpp
// In idle() callback
animHeadRot += 0.1f;
if (animHeadRot > 360.0f) animHeadRot -= 360.0f;

// In drawCharacter()
if (idx == 0) {
    modelHead = modelHead * Rotate(animHeadRot, vec3(0.0f, 1.0f, 0.0f));
}
```

#### 3.2 Character 2: Bounce Animation

상승과 하강이 반복되는 점프 애니메이션을 구현했습니다. boolean 플래그로 상태를 전환하며 Y축 이동을 제어합니다.

```cpp
// In idle() callback
if (isJump) {
    animJump += jumpSpeed;
    if (animJump >= jumpMaxHeight) isJump = false;
} else {
    animJump -= jumpSpeed;
    if (animJump <= 0.0f) isJump = true;
}

// In drawCharacter()
if (idx == 1) {
    animatedTransform = animatedTransform * Translate(0.0f, animJump, 0.0f);
}
```

#### 3.3 Character 3: Patrol with Auto-Rotation

Z축 방향으로 왕복 이동하며, 방향 전환 시 자동으로 180도 회전하여 항상 전진 방향을 바라보도록 구현했습니다.

```cpp
// In idle() callback
if (isMovingForward) {
    movePos += moveSpeed;
    if (movePos >= 2.0f) isMovingForward = false;
} else {
    movePos -= moveSpeed;
    if (movePos <= -2.0f) isMovingForward = true;
}

// In drawCharacter()
if (idx == 2) {
    animatedTransform = animatedTransform * Translate(0.0f, 0.0f, movePos);
    if (!isMovingForward) {
        // 180-degree Y-axis rotation when moving backward
        animatedTransform = animatedTransform 
                          * Rotate(M_PI, vec3(0.0f, 1.0f, 0.0f));
    }
}
```

---

### 4. Scene Setup

3개의 캐릭터와 환경 오브젝트를 배치하여 완성된 3D 씬을 구성했습니다.

```cpp
// Ground plane: 15x5x15 scale, Y=-3.0 offset
void drawGround(mat4 view, mat4 proj) {
    mat4 modelGround = model * Translate(0.0f, -3.0f, 0.0f) 
                             * Scale(15.0f, 5.0f, 15.0f);
    drawCube(VAO_ground, modelGround, view, proj);
}

// Stone blocks: 3 instances at predefined positions
vec3 stonePos[] = {
    vec3(3.0f, 0.0f, 5.0f),
    vec3(-3.5f, 0.0f, -4.0f),
    vec3(5.0f, 0.0f, -2.5f)
};

void drawStone(mat4 view, mat4 proj) {
    for (int i = 0; i < 3; i++) {
        mat4 modelStone = model * Translate(stonePos[i]) 
                                * Scale(1.0f, 1.0f, 1.0f);
        drawCube(VAO_stone, modelStone, view, proj);
    }
}
```

---

### 5. Virtual Trackball Camera (From Scratch)

Gimbal lock 없는 자연스러운 3D 회전을 위해 Virtual Trackball 알고리즘을 처음부터 구현했습니다. 화면 좌표를 구면 좌표로 변환하고, Rodriguez rotation formula를 직접 구현하여 임의의 축 기준 회전을 처리합니다.

#### 5.1 Spherical Coordinate Mapping

마우스의 2D 화면 좌표를 3D 구면 좌표로 변환하는 함수를 구현했습니다.

```cpp
void trackball_ptov(int x, int y, int width, int height, float v[3])
{
    // Normalize screen coordinates to [-1, 1]
    v[0] = (2.0f * x - width) / width;
    v[1] = (height - 2.0f * y) / height;
    
    // Calculate Z coordinate on unit sphere
    float d = sqrt(v[0] * v[0] + v[1] * v[1]);
    v[2] = cos((M_PI / 2.0f) * ((d < 1.0f) ? d : 1.0f));
    
    // Normalize to unit vector
    float a = 1.0f / sqrt(v[0]*v[0] + v[1]*v[1] + v[2]*v[2]);
    v[0] *= a; v[1] *= a; v[2] *= a;
}

void mouseMotion(int x, int y)
{
    if (isRotate) {
        float curPos1[3], curPos2[3];
        trackball_ptov(prevRotX, prevRotY, winW, winH, curPos1);
        trackball_ptov(x, y, winW, winH, curPos2);
        
        vec3 v1(curPos1[0], curPos1[1], curPos1[2]);
        vec3 v2(curPos2[0], curPos2[1], curPos2[2]);
        
        // Rotation angle: arccos of dot product
        float angle = acos(clamp(dot(v1, v2), -1.0f, 1.0f));
        // Rotation axis: cross product
        vec3 axis = normalize(cross(v1, v2));
        
        angle *= 0.5f;  // Damping factor
        mat4 rot = Rotate(angle, axis);
        model = rot * model;
        
        prevRotX = x;
        prevRotY = y;
    }
}
```

#### 5.2 Rodriguez Rotation Formula (Custom Implementation)

임의의 축을 기준으로 회전하는 4x4 변환 행렬을 생성하는 함수를 직접 구현했습니다.

```cpp
mat4 Rotate(float angle, const vec3& axis)
{
    float c = cos(angle);
    float s = sin(angle);
    float t = 1.0f - c;
    
    float x = axis.x, y = axis.y, z = axis.z;
    
    // Rodriguez formula: R = I + sin(θ)K + (1-cos(θ))K²
    return mat4(
        t*x*x + c,     t*x*y - s*z,   t*x*z + s*y,   0.0f,
        t*x*y + s*z,   t*y*y + c,     t*y*z - s*x,   0.0f,
        t*x*z - s*y,   t*y*z + s*x,   t*z*z + c,     0.0f,
        0.0f,          0.0f,          0.0f,          1.0f
    );
}
```

#### 5.3 Translation Control (Right Mouse Button)

```cpp
if (isTranslate) {
    float dx = (prevTranslateX - x) / (float)glutGet(GLUT_WINDOW_WIDTH);
    float dy = (y - prevTranslateY) / (float)glutGet(GLUT_WINDOW_HEIGHT);
    
    float camMoveSpeed = 5.0f;
    curTranslateX += dx * camMoveSpeed;
    curTranslateY += dy * camMoveSpeed;
    
    prevTranslateX = x;
    prevTranslateY = y;
}
```

#### 5.4 Zoom Control (Middle Mouse Button)

```cpp
if (isZoom) {
    float dy = (prevZoomY - y) / (float)glutGet(GLUT_WINDOW_HEIGHT);
    vec4 forward = normalize(at - eye);
    
    float curDist = length(eye - at) - dy * 7.0f;
    curDist = clamp(curDist, 1.0f, 50.0f);  // Distance constraints
    
    eye = at - forward * curDist;
    prevZoomY = y;
}
```

---

### 6. Environment Mapping Implementation

게임에서 사용되는 Skybox와 반사/굴절 효과를 직접 구현했습니다. 6개의 BMP 이미지를 큐브맵으로 로딩하는 함수부터 굴절 셰이더까지 전부 직접 작성했습니다.

#### 6.1 Cubemap Loading System

6개 방향(+X, -X, +Y, -Y, +Z, -Z)의 이미지를 하나의 GL_TEXTURE_CUBE_MAP 객체로 통합하는 로더를 구현했습니다.

```cpp
GLuint loadCubemap(const char* right, const char* left, 
                   const char* top, const char* bottom,
                   const char* front, const char* back)
{
    GLuint textureID;
    glGenTextures(1, &textureID);
    glBindTexture(GL_TEXTURE_CUBE_MAP, textureID);
    
    const char* faces[6] = {right, left, top, bottom, front, back};
    
    for (GLuint i = 0; i < 6; i++) {
        uchar4* data;
        int width, height;
        LoadBMPFile(&data, &width, &height, faces[i]);
        
        glTexImage2D(GL_TEXTURE_CUBE_MAP_POSITIVE_X + i, 
                     0, GL_RGBA, width, height, 
                     0, GL_RGBA, GL_UNSIGNED_BYTE, data);
        delete[] data;
    }
    
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_S, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_T, GL_CLAMP_TO_EDGE);
    glTexParameteri(GL_TEXTURE_CUBE_MAP, GL_TEXTURE_WRAP_R, GL_CLAMP_TO_EDGE);
    
    return textureID;
}
```

#### 6.2 Skybox Rendering with Infinite Perspective

Skybox가 카메라 이동에 영향받지 않도록 view 행렬의 translation 성분을 제거하는 트릭을 구현했습니다. depth buffer 설정으로 항상 가장 뒤에 렌더링되도록 처리했습니다.

```cpp
void renderSkybox(mat4 modelView, mat4 proj)
{
    glUseProgram(p[1]);  // Skybox shader program
    
    // Remove translation from view matrix
    mat4 skyboxView = mat4(
        vec4(modelView[0][0], modelView[0][1], modelView[0][2], 0.0f),
        vec4(modelView[1][0], modelView[1][1], modelView[1][2], 0.0f),
        vec4(modelView[2][0], modelView[2][1], modelView[2][2], 0.0f),
        vec4(0.0f, 0.0f, 0.0f, 1.0f)
    );
    
    glUniformMatrix4fv(glGetUniformLocation(p[1], "u_View"), 1, GL_TRUE, skyboxView);
    glUniformMatrix4fv(glGetUniformLocation(p[1], "u_Proj"), 1, GL_TRUE, proj);
    
    glBindVertexArray(VAO_skybox);
    glActiveTexture(GL_TEXTURE0);
    glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);
    
    // Render skybox behind all objects
    glDepthFunc(GL_LEQUAL);
    glDepthMask(GL_FALSE);
    glDrawArrays(GL_TRIANGLES, 0, 36);
    glDepthMask(GL_TRUE);
    glDepthFunc(GL_LESS);
    
    glBindVertexArray(0);
}
```

**Skybox Vertex Shader (skybox.vert)**
```glsl
#version 140

in vec3 aPos;
out vec3 texCoords;

uniform mat4 u_Proj;
uniform mat4 u_View;

void main()
{
    vec4 pos = u_Proj * u_View * vec4(aPos, 1.0f);
    gl_Position = vec4(pos.x, pos.y, pos.w, pos.w);  // z = w for far plane
    texCoords = vec3(aPos.x, -aPos.y, aPos.z);
}
```

**Skybox Fragment Shader (skybox.frag)**
```glsl
#version 140

in vec3 texCoords;
out vec4 FragColor;

uniform samplerCube skybox;

void main()
{    
    FragColor = texture(skybox, texCoords);
}
```

#### 6.3 Refraction Shader (Glass Effect)

유리/거울 재질을 표현하기 위해 굴절 셰이더를 작성했습니다. Normal Matrix를 사용한 법선 변환과 GLSL의 refract() 함수로 물리적으로 정확한 굴절을 계산합니다.

```cpp
// Normal vector data for each cube face
float mCubeNormals[] = {
    // Front face
    0.0f,  0.0f,  1.0f,  0.0f,  0.0f,  1.0f,
    0.0f,  0.0f,  1.0f,  0.0f,  0.0f,  1.0f,
    // Back face
    0.0f,  0.0f, -1.0f,  0.0f,  0.0f, -1.0f,
    // ... (left, right, top, bottom faces)
};

// Mirror cube rendering
glUseProgram(p[2]);  // Mirror shader program

mat4 mirrorTransform = model * Translate(-5.0f, 1.0f, 4.0f) 
                             * Scale(3.0f, 3.0f, 3.0f);

glUniformMatrix4fv(glGetUniformLocation(p[2], "u_Model"), 1, GL_TRUE, mirrorTransform);
glUniformMatrix4fv(glGetUniformLocation(p[2], "u_View"), 1, GL_TRUE, view);
glUniformMatrix4fv(glGetUniformLocation(p[2], "u_Proj"), 1, GL_TRUE, proj);
glUniform3f(glGetUniformLocation(p[2], "u_CameraPos"), 
            eye.x + curTranslateX, eye.y + curTranslateY, eye.z);

glActiveTexture(GL_TEXTURE0);
glBindTexture(GL_TEXTURE_CUBE_MAP, cubemapTexture);

glBindVertexArray(VAO_mirror);
glDrawElements(GL_TRIANGLES, 36, GL_UNSIGNED_INT, 0);
```

**Mirror Vertex Shader (mirror.vert)**
```glsl
#version 140

in vec3 aPos;
in vec3 aNormal;

out vec3 Normal;
out vec3 Position;

uniform mat4 u_Model;
uniform mat4 u_View;
uniform mat4 u_Proj;

void main()
{
    // Transform normal using Normal Matrix: transpose(inverse(M))
    Normal = mat3(transpose(inverse(u_Model))) * aNormal;
    Position = vec3(u_Model * vec4(aPos, 1.0));
    gl_Position = u_Proj * u_View * vec4(Position, 1.0);
}
```

**Mirror Fragment Shader (mirror.frag)**
```glsl
#version 140

out vec4 FragColor;

in vec3 Normal;
in vec3 Position;

uniform vec3 u_CameraPos;
uniform samplerCube skybox;

void main()
{
    float ratio = 1.00 / 1.52;  // Air/Glass refractive index
    vec3 I = normalize(Position - u_CameraPos);  // Incident vector
    vec3 R = refract(I, normalize(Normal), ratio);  // Refraction vector
    FragColor = vec4(texture(skybox, R).rgb, 1.0);
}
```

---

## Shader Code (Written by Me)

### Basic Texture Mapping Shader

텍스처 샘플링을 위한 기본 셰이더를 작성했습니다.

**Vertex Shader (vshader.vert)**
```glsl
#version 140
#extension GL_ARB_compatibility: enable

in vec2 aTexCoord;
out vec2 texCoord;

uniform mat4 u_MVP;

void main() 
{
   gl_Position = u_MVP * gl_Vertex; 
   texCoord = aTexCoord;
}
```

**Fragment Shader (fshader.frag)**
```glsl
#version 140
#extension GL_ARB_compatibility: enable

in vec2 texCoord;
out vec4 fColor;

uniform sampler2D u_Tex;

void main() 
{ 
    fColor = texture(u_Tex, texCoord);
}
```

---

## Implementation Details

### VAO/VBO Architecture

```cpp
void initBuffers()
{
    // Shared position buffer
    glGenBuffers(1, &VBO_pos);
    glBindBuffer(GL_ARRAY_BUFFER, VBO_pos);
    glBufferData(GL_ARRAY_BUFFER, sizeof(vPos), vPos, GL_STATIC_DRAW);
    
    // Shared index buffer
    glGenBuffers(1, &IBO);
    glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, IBO);
    glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);
    
    // Per-character VAO/VBO setup
    for (int i = 0; i < 3; i++) {
        // Head
        glGenVertexArrays(1, &VAO_head[i]);
        glBindVertexArray(VAO_head[i]);
        
        // Position attribute (location=0)
        glBindBuffer(GL_ARRAY_BUFFER, VBO_pos);
        glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3*sizeof(float), 0);
        glEnableVertexAttribArray(0);
        
        // Texture coordinate attribute (location=1)
        glGenBuffers(1, &VBO_head[i]);
        glBindBuffer(GL_ARRAY_BUFFER, VBO_head[i]);
        glBufferData(GL_ARRAY_BUFFER, sizeof(texHead), texHead, GL_STATIC_DRAW);
        glVertexAttribPointer(1, 2, GL_FLOAT, GL_FALSE, 2*sizeof(float), 0);
        glEnableVertexAttribArray(1);
        
        glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, IBO);
        
        // Body, LeftArm, RightArm, Leg similarly configured
    }
    
    // Environment objects (Ground, Stone) VAO setup
    // Skybox VAO setup
    // Mirror cube VAO setup with position and normal attributes
}
```

### Rendering Pipeline

```cpp
void renderScene(void)
{
    glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
    
    // View and projection matrices
    mat4 view = LookAt(eye + vec4(curTranslateX, curTranslateY, 0, 0),
                       at + vec4(curTranslateX, curTranslateY, 0, 0), up);
    mat4 proj = Perspective(fov, aspect, nearClip, farClip);
    
    // Render characters with basic texture mapping shader
    glUseProgram(p[0]);
    for (int i = 0; i < 3; i++) {
        glBindTexture(GL_TEXTURE_2D, color_tex[i]);
        glUniform1i(glGetUniformLocation(p[0], "u_Tex"), 0);
        
        mat4 moveCharacter = Translate(-2.0f + 2.0f * i, 0.0f, 0.0f);
        drawCharacter(model * moveCharacter, view, proj, i);
    }
    
    // Render environment objects
    glBindTexture(GL_TEXTURE_2D, color_tex[3]);
    drawGround(view, proj);
    
    glBindTexture(GL_TEXTURE_2D, color_tex[4]);
    drawStone(view, proj);
    
    // Render skybox with cubemap shader
    renderSkybox(model, proj);
    
    // Render mirror cube with refraction shader
    glUseProgram(p[2]);
    // Mirror rendering code...
    
    glutSwapBuffers();
}
```


### 구현한 것들
- 버추얼 트랙볼
- 벡터 회전 (임의 축 기준 회전)
- 큐브 맵 환경 매핑
- 반사 벡터 구현
- 캐릭터 애니메이션 세 가지

- 정점 위치 데이터는 공유 VBO 1개로 통합 (메모리 절약)
- 텍스처 좌표만 캐릭터별로 분리하여 유연성 확보
- Element Buffer Object로 인덱싱하여 정점 재사용
- Skybox는 depth buffer 설정으로 overdraw 방지

---

## 트러블슈팅

1. 페이퍼크래프트 이미지의 240개 UV 좌표를 수작업으로 추출하면서 텍스처 공간의 구조를 완전히 이해하게 되었습니다. 각 픽셀 위치를 [0,1] 범위로 정규화하고, 큐브 전개도의 어느 면이 3D 공간의 어느 방향에 대응되는지 매핑하는 과정이 필요했습니다. 수동 과정이 굉장히 비효율적이라고 생각이 들었지만 일단 진행했습니다.

2. 반사/굴절 효과 구현 중 비균등 스케일 변환 시 법선 벡터가 왜곡되는 문제를 발견했습니다. transpose(inverse(Model)) 연산으로 Normal Matrix를 생성하여 해결했습니다.

3. 처음엔 단순한 X/Y축 회전으로 카메라를 구현했으나 짐볼 락 문제가 발생했습니다. Virtual Trackball으로 교체한 후 훨씬 자연스러운 3D 회전이 가능해졌고, Rodriguez formula 공식을 검색해 가져왔습니다.

4. Skybox가 카메라와 함께 움직이는 버그를 겪었습니다. View 행렬의 4번째 열(translation)을 제거하고 vertex shader에서 z=w 처리를 하니 해결되었습니다. 이런 트릭들이 실제 게임 엔진에서도 사용된다는 점이 흥미로웠습니다.

---
