# Project ECHO: Game Engine Architecture

## Table of Contents

1. [Rendering Pipeline](#rendering-pipeline)
2. [Entity-Component-System](#entity-component-system)
3. [Physics Engine](#physics-engine)
4. [Animation System](#animation-system)
5. [Audio Engine](#audio-engine)
6. [Particle Effects](#particle-effects)
7. [World Management](#world-management)
8. [Performance Optimization](#performance-optimization)

---

## Rendering Pipeline

### Multi-API Support (Vulkan + OpenGL)

```cpp
class Renderer {
    enum class API {
        Vulkan,   // Preferred (modern, explicit)
        OpenGL,   // Fallback (broader compatibility)
    };
    
private:
    API api_type;
    std::unique_ptr<RenderBackend> backend;
    
public:
    Renderer(API api) : api_type(api) {
        if (api == API::Vulkan) {
            backend = std::make_unique<VulkanBackend>();
        } else {
            backend = std::make_unique<OpenGLBackend>();
        }
    }
    
    void begin_frame() {
        backend->begin_frame();
    }
    
    void render_entity(const Entity& entity) {
        backend->render_mesh(
            entity.get<RenderableComponent>().mesh,
            entity.get<TransformComponent>().transform_matrix,
            entity.get<RenderableComponent>().material
        );
    }
    
    void end_frame() {
        backend->present();
    }
};
```

### Vulkan Pipeline Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  Depth Pre-Pass (early-z)              │
│  Render depth only, cull fragments     │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  G-Buffer Pass (deferred)              │
│  Position, Normal, Albedo, Roughness  │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  Lighting Pass                         │
│  Point, directional, ambient lights   │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  Transparency Pass                     │
│  Alpha blending, particles             │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  Post-Processing                       │
│  Bloom, FXAA, color grading            │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  UI Rendering                         │
│  HUD, menus, debug info               │
└──────────────────────┬──────────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────────┐
│  Frame Present                        │
│  Swap buffers, vsync                  │
└──────────────────────────────────────────────────────────────┘
```

---

## Entity-Component-System

### Core ECS Architecture

```cpp
// Base component
class Component {
public:
    virtual ~Component() = default;
};

// Specific components
struct TransformComponent : public Component {
    Vec3 position;
    Quaternion rotation;
    Vec3 scale;
};

struct RenderableComponent : public Component {
    Mesh* mesh;
    Material* material;
    std::vector<Texture*> textures;
};

struct PhysicsComponent : public Component {
    btRigidBody* rigid_body;
    Vec3 velocity;
    Vec3 angular_velocity;
};

struct AnimationComponent : public Component {
    AnimationController* controller;
    std::string current_animation;
    float animation_time;
};

// Entity
class Entity {
    UUID id;
    std::unordered_map<std::type_index, std::unique_ptr<Component>> components;
    
public:
    template<typename T, typename... Args>
    T& add_component(Args&&... args) {
        auto component = std::make_unique<T>(std::forward<Args>(args)...);
        auto& ref = *component;
        components[std::type_index(typeid(T))] = std::move(component);
        return ref;
    }
    
    template<typename T>
    T& get_component() {
        auto it = components.find(std::type_index(typeid(T)));
        if (it == components.end()) {
            throw std::runtime_error("Component not found");
        }
        return *static_cast<T*>(it->second.get());
    }
    
    template<typename T>
    bool has_component() const {
        return components.find(std::type_index(typeid(T))) != components.end();
    }
};

// System (operates on components)
class System {
public:
    virtual ~System() = default;
    virtual void update(World& world, float dt) = 0;
};

// World
class World {
    std::unordered_map<UUID, Entity> entities;
    std::vector<std::unique_ptr<System>> systems;
    
public:
    Entity& spawn_entity() {
        UUID id = generate_uuid();
        return entities[id];
    }
    
    void update(float dt) {
        for (auto& system : systems) {
            system->update(*this, dt);
        }
    }
};
```

### System Examples

```cpp
class PhysicsSystem : public System {
    btDynamicsWorld* dynamics_world;
    
public:
    void update(World& world, float dt) override {
        // Step physics
        dynamics_world->stepSimulation(dt);
        
        // Update transform components from physics
        for (auto& [id, entity] : world.entities) {
            if (entity.has_component<PhysicsComponent>() &&
                entity.has_component<TransformComponent>()) {
                
                auto& physics = entity.get_component<PhysicsComponent>();
                auto& transform = entity.get_component<TransformComponent>();
                
                // Get rigid body transform
                btTransform bt_transform;
                physics.rigid_body->getMotionState()->getWorldTransform(bt_transform);
                
                // Update entity position/rotation
                transform.position = to_vec3(bt_transform.getOrigin());
                transform.rotation = to_quaternion(bt_transform.getRotation());
            }
        }
    }
};

class AnimationSystem : public System {
public:
    void update(World& world, float dt) override {
        for (auto& [id, entity] : world.entities) {
            if (entity.has_component<AnimationComponent>()) {
                auto& anim = entity.get_component<AnimationComponent>();
                anim.animation_time += dt;
                anim.controller->update(dt);
            }
        }
    }
};

class RenderSystem : public System {
public:
    void update(World& world, float dt) override {
        // Rendering happens in separate render thread
    }
    
    void render(Renderer& renderer, World& world) {
        renderer.begin_frame();
        
        for (auto& [id, entity] : world.entities) {
            if (entity.has_component<RenderableComponent>() &&
                entity.has_component<TransformComponent>()) {
                renderer.render_entity(entity);
            }
        }
        
        renderer.end_frame();
    }
};
```

---

## Physics Engine

### Bullet3 Integration

```cpp
class PhysicsWorld {
    btDefaultCollisionConfiguration* collision_config;
    btCollisionDispatcher* dispatcher;
    btBroadphaseInterface* broadphase;
    btSequentialImpulseConstraintSolver* solver;
    btDynamicsWorld* dynamics_world;
    
public:
    PhysicsWorld() {
        collision_config = new btDefaultCollisionConfiguration();
        dispatcher = new btCollisionDispatcher(collision_config);
        broadphase = new btDbvtBroadphase();
        solver = new btSequentialImpulseConstraintSolver();
        
        dynamics_world = new btDiscreteDynamicsWorld(
            dispatcher, broadphase, solver, collision_config);
        dynamics_world->setGravity(btVector3(0, -9.81f, 0));
    }
    
    btRigidBody* create_rigid_body(
        float mass,
        const Vec3& position,
        const btCollisionShape* shape
    ) {
        btTransform transform;
        transform.setIdentity();
        transform.setOrigin(btVector3(position.x, position.y, position.z));
        
        btMotionState* motion_state = new btDefaultMotionState(transform);
        
        btRigidBody::btRigidBodyConstructionInfo rbInfo(
            mass, motion_state, shape);
        
        btRigidBody* body = new btRigidBody(rbInfo);
        dynamics_world->addRigidBody(body);
        
        return body;
    }
    
    void step(float dt) {
        dynamics_world->stepSimulation(dt, 10);  // 10 substeps
    }
};
```

---

## Animation System

### Skeletal Animation

```cpp
class Skeleton {
    struct Bone {
        std::string name;
        int parent_index;
        Matrix4 bind_pose;
        Matrix4 current_pose;
    };
    
    std::vector<Bone> bones;
    
public:
    void update_bone(int bone_index, const Matrix4& transform) {
        bones[bone_index].current_pose = transform;
    }
    
    const std::vector<Matrix4> get_bone_matrices() {
        std::vector<Matrix4> matrices;
        for (const auto& bone : bones) {
            matrices.push_back(bone.current_pose);
        }
        return matrices;
    }
};

class AnimationClip {
    std::string name;
    float duration;
    std::vector<AnimationTrack> tracks;  // One per bone
    
public:
    void sample(float time, Skeleton& skeleton) {
        for (size_t i = 0; i < tracks.size(); ++i) {
            auto transform = tracks[i].sample(time);
            skeleton.update_bone(i, transform);
        }
    }
};
```

---

## Audio Engine

### 3D Spatial Audio

```cpp
class AudioEngine {
    ALCdevice* device;
    ALCcontext* context;
    std::unordered_map<UUID, ALuint> sources;
    
public:
    void set_listener_position(const Vec3& pos, const Vec3& forward, const Vec3& up) {
        alListener3f(AL_POSITION, pos.x, pos.y, pos.z);
        
        ALfloat orientation[] = {
            forward.x, forward.y, forward.z,
            up.x, up.y, up.z
        };
        alListerfv(AL_ORIENTATION, orientation);
    }
    
    ALuint create_source(const UUID& entity_id, const Vec3& position) {
        ALuint source;
        alGenSources(1, &source);
        
        alSource3f(source, AL_POSITION, position.x, position.y, position.z);
        alSourcef(source, AL_REFERENCE_DISTANCE, 1.0f);
        alSourcef(source, AL_MAX_DISTANCE, 1000.0f);
        
        sources[entity_id] = source;
        return source;
    }
    
    void update_source_position(const UUID& entity_id, const Vec3& new_pos) {
        auto it = sources.find(entity_id);
        if (it != sources.end()) {
            alSource3f(it->second, AL_POSITION, new_pos.x, new_pos.y, new_pos.z);
        }
    }
};
```

---

## Particle Effects

### Particle System

```cpp
class ParticleSystem {
    struct Particle {
        Vec3 position;
        Vec3 velocity;
        float lifetime;
        float age;
    };
    
    std::vector<Particle> particles;
    size_t max_particles = 10000;
    
public:
    void emit(const Vec3& position, const Vec3& velocity) {
        if (particles.size() < max_particles) {
            particles.push_back({
                position,
                velocity,
                2.0f,  // Lifetime in seconds
                0.0f
            });
        }
    }
    
    void update(float dt) {
        for (auto it = particles.begin(); it != particles.end();) {
            it->age += dt;
            it->position += it->velocity * dt;
            it->velocity *= 0.99f;  // Air friction
            
            if (it->age >= it->lifetime) {
                it = particles.erase(it);
            } else {
                ++it;
            }
        }
    }
    
    void render(Renderer& renderer) {
        for (const auto& particle : particles) {
            float alpha = 1.0f - (particle.age / particle.lifetime);
            renderer.render_billboard(particle.position, alpha);
        }
    }
};
```

---

## World Management

### Spatial Partitioning

```cpp
class Octree {
    struct Node {
        Vec3 min, max;
        std::vector<UUID> entities;
        std::array<std::unique_ptr<Node>, 8> children;
    };
    
    Node root;
    
public:
    void insert(const UUID& entity_id, const Vec3& position) {
        insert_recursive(&root, entity_id, position, 0);
    }
    
    std::vector<UUID> query_sphere(const Vec3& center, float radius) {
        std::vector<UUID> results;
        query_sphere_recursive(&root, center, radius, results);
        return results;
    }
    
private:
    void insert_recursive(Node* node, const UUID& entity_id, const Vec3& pos, int depth) {
        if (node->entities.size() < MAX_ENTITIES_PER_NODE || depth >= MAX_DEPTH) {
            node->entities.push_back(entity_id);
        } else {
            // Subdivide
            int child_index = get_octant(node->min, node->max, pos);
            if (!node->children[child_index]) {
                node->children[child_index] = std::make_unique<Node>();
                // Set bounds
            }
            insert_recursive(node->children[child_index].get(), entity_id, pos, depth + 1);
        }
    }
};
```

---

## Performance Optimization

### Frustum Culling

```cpp
class Camera {
    Matrix4 view_matrix;
    Matrix4 projection_matrix;
    
public:
    bool is_in_frustum(const Vec3& position, float radius) const {
        // Transform to camera space
        Vec4 cam_space = view_matrix * Vec4(position, 1.0f);
        
        // Check against frustum planes
        // ... plane intersection tests ...
        
        return true;
    }
};
```

### Level of Detail (LOD)

```cpp
class LODManager {
    struct LODLevel {
        Mesh* mesh;
        float distance_threshold;
    };
    
    std::vector<LODLevel> lod_levels;
    
public:
    Mesh* get_lod_mesh(const Vec3& camera_pos, const Vec3& entity_pos) {
        float distance = vec3_distance(camera_pos, entity_pos);
        
        for (const auto& level : lod_levels) {
            if (distance < level.distance_threshold) {
                return level.mesh;
            }
        }
        
        return lod_levels.back().mesh;  // Lowest LOD
    }
};
```

### Batch Rendering

```cpp
class RenderBatch {
    Material* material;
    std::vector<std::pair<Mesh*, Matrix4>> instances;
    
public:
    void add_instance(Mesh* mesh, const Matrix4& transform) {
        instances.push_back({mesh, transform});
    }
    
    void render(Renderer& renderer) {
        // Submit all instances with same material in one draw call
        renderer.render_batch(material, instances);
    }
};
```

---

## Summary

Project ECHO's game engine provides:
✅ Dual rendering (Vulkan + OpenGL)
✅ High-performance ECS architecture
✅ Physics integration (Bullet3)
✅ Skeletal animation support
✅ 3D spatial audio (OpenAL)
✅ Particle effects system
✅ Spatial partitioning (Octree)
✅ Optimizations (frustum culling, LOD, batching)
