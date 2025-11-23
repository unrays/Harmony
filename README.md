# Harmony
A C++ sound engine focused on orchestrating multiple sound layers efficiently

This is my first time experimenting with namespaces in C++, and I've tried to create an architecture that I think is very interesting. Now it's up to you to judge; I'm open to all your feedback. Oh, by the way, I also tried working with the miniaudio library to help me conceptualize and build everything. It's far from being finished :)

This is the first time in my life I've attempted to design a sound engine, and I find it incredibly exciting and addictive ;) Furthermore, it's the first time I've worked with namespaces, and I find them extremely useful. They allow you to add a fourth dimension to the architecture, much like a file tree would. Although it's not finished yet, I still want to say that I'm really satisfied with the progress I've made.

*Due to my lack of knowledge and time, I think I'll take a break from this project. Maybe I'll come back to it someday, who knows? In the meantime, keep an eye out for new updates on my GitHub account ;)*
```cpp
// Copyright (c) November 2025 Félix-Olivier Dumas. All rights reserved.
// Licensed under the terms described in the LICENSE file

namespace Audio {
    namespace Internal {
        struct Flags {
            enum class Status : uint8_t {
                Off = 0,
                On = 1,
                Starting = 2,
                Pending = 3,
                Error = 4
            };
            enum class State : std::uint8_t {
                None = 0,
                Stopped = 1 << 0,
                Playing = 1 << 1,
                Paused = 1 << 2,
                Released = 1 << 3
            };
            enum class Error : std::uint8_t {
                None = 0,
                Warning = 1 << 0,
                Crash = 1 << 1,
                Fatal = 1 << 2
            };
        };
        struct Utilities {
        private:
            using InternalFlags = Audio::Internal::Flags;
            using StateEnum = InternalFlags::State;
            using StateError = InternalFlags::Error;
        public:
            struct Flags {
            private:
            public:
                static bool IS_VALID_STATE(StateEnum s) {
                    return s == StateEnum::None
                        || s == StateEnum::Stopped
                        || s == StateEnum::Playing
                        || s == StateEnum::Paused;
                }
                static bool IS_VALID_ERROR(StateError e) {
                    return e == StateError::None
                        || e == StateError::Warning
                        || e == StateError::Crash
                        || e == StateError::Fatal;
                }
            };
            struct Playback {
            private:
            public:
                template<typename Func>
                bool CAN_PLAY(Func getState) {
                    static_assert(std::is_same_v<decltype(getState()), StateEnum>,
                        "Function must return StateEnum");
                    StateEnum state = getState();
                    return !(static_cast<uint8_t>(state)
                        & (static_cast<uint8_t>(StateEnum::Paused)
                        |  static_cast<uint8_t>(StateEnum::Stopped)
                        |  static_cast<uint8_t>(StateEnum::None)));
                }
            };
        };
    } 
    using Types = Internal::Flags;
    using Utils = Internal::Utilities;
    
    namespace Settings {
        inline float& Volume = Configuration::Audio::Volume;
    }
    class Engine {
    private:
        struct Stream {
            Internal::Flags::State currentState;
            uint32_t currentFrame;
            float& currentVolume;
            uint8_t currentNote;

            Stream() : currentState(Internal::Flags::State::Stopped),
                currentFrame(0), currentVolume(Settings::Volume),
                currentNote(0) { }
        };
        struct Playback {
        private:
            Stream stream;

        public:
            /*Un peu boilerplate ici, sert pas a grand chose*/
            void SetCurrentNote(const uint8_t note) {
                if (note <= 127) stream.currentNote = note;
            }
            void SetCurrentFrame(const uint32_t frame) {
                stream.currentFrame = frame;
            }
            void SetCurrentState(const Types::State state) {
                if (Internal::Utilities::Flags::IS_VALID_STATE(state))
                    stream.currentState = state;
            }
            void SetCurrentVolume(const float volume) {
                if (volume <= 1) stream.currentVolume = volume;
            }

            auto GetCurrentNote() { return stream.currentNote; }
            auto GetCurrentFrame() { return stream.currentFrame; }
            auto GetCurrentState() { return stream.currentState; }
            auto GetCurrentVolume() { return stream.currentVolume; }
        }; 
        using Status = Audio::Internal::Flags::Status;
        Status status;
        Playback player;

    private:
        Engine(std::unique_ptr<Playback> initPlayer)
            : player(std::move(initPlayer)), status(Status::Off) {}
    
        void onStarting() { status = Status::Starting; }
        void onCreated() { status = Status::On; }
    
        bool is(Status s) const { return status == s; }
    
    public:
        static Engine create() {
            auto instance = Engine(std::make_unique<Playback>());
            instance.onStarting();
            instance.onCreated();
            return instance;
        }
        
        bool isOn() const { return is(Status::On);; }
        bool isOff() const { return is(Status::Off);; }
    };
}

auto engine = Audio::Engine::create();
```
