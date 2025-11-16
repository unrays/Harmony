# Harmony
A C++ sound engine focused on orchestrating multiple sound layers efficiently


```cpp
// Copyright (c) November 2025 Félix-Olivier Dumas. All rights reserved.
// Licensed under the terms described in the LICENSE file

#define MINIAUDIO_IMPLEMENTATION
#include "miniaudio.h"
#include <iostream>
#include <cstdint>
#include <thread>
#include <chrono>
#include <type_traits>


namespace Configuration {
    namespace Graphics { }
    namespace Audio {
        inline const float Volume = 0.05f;
    }
}

//Exemple : C5 est 3 demi-tons au-dessus de A4 → 𝑛=3n = 3, donc𝑓=440∗23/12f = 440∗23 / 12

namespace Audio {
    enum class State : uint8_t {
        None = 0,
        Stopped = 1 << 0,
        Playing = 1 << 1,
        Paused = 1 << 2,
        Released = 1 << 3
    };

    namespace Utils {
        static bool IS_VALID_STATE(State s) {
            return s == State::None || s == State::Stopped || s == State::Playing || s == State::Paused;
        }
    };

    class Playback {
    private:
        struct InternalState {
            State currentState = State::Stopped;
            uint32_t currentFrame = 0;
            float currentVolume = 0.05f;
            uint8_t currentNote = 0;
        };
        inline static InternalState internal;

        Playback() = delete;
        ~Playback() = delete;

        Playback(const Playback&) = delete;
        Playback& operator=(const Playback&) = delete;
        Playback(Playback&&) = delete;
        Playback& operator=(Playback&&) = delete;

    public:
        static auto GetCurrentNote() { return internal.currentNote; }
        static auto GetCurrentFrame() { return internal.currentFrame; }
        static auto GetCurrentState() { return internal.currentState; }
        static auto GetCurrentVolume() { return internal.currentVolume; }

        static bool IS_ALLOWED_TO_PLAY() {
            return !(static_cast<uint8_t>(internal.currentState)
                & (static_cast<uint8_t>(State::Paused)
                    | static_cast<uint8_t>(State::Stopped)
                    | static_cast<uint8_t>(State::None)));
        }

    private:
        static void SetCurrentNote(const uint8_t note) {
            if (note <= 127) internal.currentNote = note;
        }
        static void SetCurrentFrame(const uint32_t frame) { internal.currentFrame = frame; }
        static void SetCurrentState(const State state) {
            if (Utils::IS_VALID_STATE(state)) internal.currentState = state;
        }
        static void SetCurrentVolume(const float volume) {
            if (volume <= 1) internal.currentVolume = volume;
        }

        friend class Engine;
    };

    class Engine {
    private:
        Engine() = delete;
        ~Engine() = delete;

        Engine(const Playback&) = delete;
        Engine& operator=(const Engine&) = delete;
        Engine(Engine&&) = delete;
        Engine& operator=(Engine&&) = delete;

    public:
        static void playSound(uint8_t freq, size_t ms) {
            std::thread([freq, ms]() {
                Audio::Playback::SetCurrentState(State::Playing);
                std::this_thread::sleep_for(std::chrono::milliseconds(ms));
                Audio::Playback::SetCurrentState(State::Stopped);
                }).detach();
        }
    };

    struct Stream { static void NotImplemented() {} };

    namespace Internal {

        void testPlaySound(const uint8_t freq, const size_t milliseconds) {
            Audio::Engine::playSound(freq, milliseconds);
        }

        void onAudioCallback(ma_device* pDevice, void* pOutput, const void* pInput, ma_uint32 frameCount) {
            if (!Audio::Playback::IS_ALLOWED_TO_PLAY()) return;

            static unsigned int phase = 0; // conserve la phase entre les appels
            unsigned char* out = (unsigned char*)pOutput;

            /*playback.playsound ou whatever avec surcharge qui supporte les notes*/

            for (ma_uint32 i = 0; i < frameCount; ++i) {
                // frequence = environ 440 Hz, sample rate = 44100
                // on divise par 50 pour avoir une freq audible (environ 440Hz)
                float volume = Audio::Playback::GetCurrentVolume(); // 0 = silence, 1 = max
                out[i] = 128 + (unsigned char)((((phase / 50) % 2 == 0) ? 127 : -127) * volume);
                phase++;
            }
        }
    }
}



int main() {
    ma_context context;

    ma_result result;
    result = ma_context_init(NULL, 0, NULL, &context);

    if (result != MA_SUCCESS) {
        printf("Error while initializing audio context.\n");
        return -1;
    }

    /********************************************************/

    ma_device_info* pPlaybackDevices = nullptr;
    ma_uint32 playbackDeviceCount = 0;

    ma_result resultEnum = ma_context_get_devices(&context,
        &pPlaybackDevices,
        &playbackDeviceCount,
        nullptr,
        nullptr);

    for (ma_uint32 i = 0; i < playbackDeviceCount; ++i) {
        printf("Device #%d: %s\n", i, pPlaybackDevices[i].name);
    }

    /***************************************************************/

    ma_device device;        
    ma_device_config config;

    config = ma_device_config_init(ma_device_type_playback);
    config.playback.format = ma_format_u8;   // 8-bit
    config.playback.channels = 1;            // mono
    config.sampleRate = 44100;               // standard
    config.dataCallback = Audio::Internal::onAudioCallback;

    result = ma_device_init(&context, &config, &device);  // init
    if (result != MA_SUCCESS) {
        printf("Erreur: impossible d'initialiser le device audio.\n");
        return -1;
    }

    ma_device_start(&device);


    Audio::Engine::playSound(440, 1000);
    printf("Appuie sur Entrée pour quitter...\n");
    std::cin.get();

    /*********************************************************************/

    //Audio::Synth::playSound(&device, 440, 0.05f);

    Audio::Stream::NotImplemented();

    auto value = Configuration::Audio::Volume;

    ma_context_uninit(&context);
}
```
