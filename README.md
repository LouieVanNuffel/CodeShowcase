# Game Systems Showcase


## Table of Contents

- [Modular input system and command-action bindings](#modular-input-system-and-command-action-bindings)
  - [Controller.h](#controllerh)
  - [XInputImpl.cpp](#xinputimplcpp)
  - [ControllerComponent.h](#controllercomponenth)
  - [Command.h](#commandh)
  - [Example of actions](#example-of-actions)
  - [Example of binding an action with a command](#example-of-binding-an-action-with-a-command)
  - [Examples of setting up controls](#examples-of-setting-up-controls)
    - [GamepadControllerComponent.h](#examples-of-setting-up-controls)
    - [KeyboardControllerComponent.h](#examples-of-setting-up-controls)
- [Implementation of observer pattern](#implementation-of-observer-pattern)
  - [Observer.h](#observerh)
  - [Subject.h](#subjecth)
  - [Event.h](#eventh)
  - [Example of using the pattern](#example-of-using-the-pattern)
    - [Simplified HealthComponent.h](#example-of-using-the-pattern)
    - [Simplified HealthDisplayComponent.h](#example-of-using-the-pattern)
- [Queued Sound System And Service Locator](#queued-sound-system-and-service-locator)
  - [SoundSystem.h](#soundsystemh)
  - [ServiceLocator.h](#servicelocatorh)
  - [ServiceLocator.cpp](#servicelocatorcpp)
  - [QueuedSoundSystem.h](#queuedsoundsystemh)
    - [QueuedSoundSystemImplementation.cpp](#queuedsoundsystemh)
    - [NullSoundSystem.h](#queuedsoundsystemh)
    - [LoggingSoundSystem.h](#queuedsoundsystemh)
  - [Usage of service locator and sound system](#usage-of-service-locator-and-sound-system)
    - [Registering](#usage-of-service-locator-and-sound-system)
    - [Using](#usage-of-service-locator-and-sound-system)



## Modular input system and command-action bindings

To have a gamepad input system that is easily adjustable when switching API's or platforms, I made a Controller singleton that can be asked for gamepad input information, whilst the actual specific implementation lives in a cpp file. The Controller simply holds a pointer to the implementation.

### Controller.h
``` C++
#pragma once
#include "Singleton.h"
#include "XInputImpl.cpp"
#include <memory>

namespace dae
{
	class Controller : public Singleton<Controller>
	{
	public:
		void Update();

		bool IsDownThisFrame(unsigned int button, int controllerIndex) const;
		bool IsUpThisFrame(unsigned int button, int controllerIndex) const;
		bool IsPressed(unsigned int button, int controllerIndex) const;

	private:
		std::unique_ptr<XInputImpl> m_pXInputImpl{ std::make_unique<XInputImpl>() };

	};
}
```

### XInputImpl.cpp

XInputImpl deliberately has no header. It is private implementation detail hidden behind Controller. This makes swapping to a different system simple without having to touch higher-level gameplay code.

<details>
<summary>Implementation</summary>

``` C++
#pragma once
#define WIN32_LEAN_AND_MEAN
#include "Windows.h"
#include "Xinput.h"

namespace dae
{
	class XInputImpl
	{
	public:
        void Update()
        {
            for (DWORD controllerIndex = 0; controllerIndex < 4; controllerIndex++)
            {
                CopyMemory(&previousState[controllerIndex], &currentState[controllerIndex], sizeof(XINPUT_STATE));
                ZeroMemory(&currentState[controllerIndex], sizeof(XINPUT_STATE));
                XInputGetState(controllerIndex, &currentState[controllerIndex]);

                buttonChanges[controllerIndex] = currentState[controllerIndex].Gamepad.wButtons 
                                                ^ previousState[controllerIndex].Gamepad.wButtons;
                buttonsPressedThisFrame[controllerIndex] = buttonChanges[controllerIndex] 
                                                        & currentState[controllerIndex].Gamepad.wButtons;
                buttonsReleasedThisFrame[controllerIndex] = buttonChanges[controllerIndex] 
                                                        & (~currentState[controllerIndex].Gamepad.wButtons);
            }
        }

        bool IsDownThisFrame(unsigned int button, int controllerIndex) const
        {
            return buttonsPressedThisFrame[controllerIndex] & button;
        }

        bool IsUpThisFrame(unsigned int button, int controllerIndex) const
        {
            return buttonsReleasedThisFrame[controllerIndex] & button;
        }

        bool IsPressed(unsigned int button, int controllerIndex) const
        {
            return currentState[controllerIndex].Gamepad.wButtons & button;
        }

    private:
        XINPUT_STATE previousState[4];
        XINPUT_STATE currentState[4];

        unsigned int buttonChanges[4];
        unsigned int buttonsPressedThisFrame[4];
        unsigned int buttonsReleasedThisFrame[4];
	};
}
```
</details>

&ensp;
---

### ControllerComponent.h

To be able to both control an object with player input and logic components for enemies we bind input to actions, and then bind actions to commands. These actions can then be directly called from other components.

``` C++
#pragma once
#include "Component.h"
#include <unordered_map>
#include "Command.h"

namespace dae
{
	class ControllerComponent : public Component
	{
	public:
		ControllerComponent(GameObject* gameObject)
			: Component(gameObject)
		{

		}
        
		virtual void Update()
        {
            for (const auto& commandActionPair : m_CommandActionBindings)
            {
                if (ActionHappened(commandActionPair.first))
                {
                    commandActionPair.second->Execute();
                }
            }
        }

		void BindCommandToAction(std::unique_ptr<Command> command, uint32_t action)
        {
            m_CommandActionBindings.emplace(action, std::move(command));
        }

		void ExecuteAction(uint32_t action)
        {
            m_CommandActionBindings[action]->Execute();
        }

	protected:
		virtual bool ActionHappened(uint32_t action) = 0;

	private:
		std::unordered_map<uint32_t, std::unique_ptr<Command>> m_CommandActionBindings;
	};
}
```

### Command.h

``` C++
#pragma once
namespace dae
{
	class Command
	{
	public:
		virtual ~Command() = default;
		virtual void Execute() = 0;
	};
}
```

### Example of actions

``` C++
enum class Action
{
	up, down, left, right, push, breakBlock, stun, takeDamage
};
```

### Example of binding an action with a command

``` C++
auto moveUpCommand = std::make_unique<TileBasedMoveCommand>(characterObject1.get(), 0.f, -1.f);
auto keyboardControllerComponent = std::make_unique<KeyboardControllerComponent>(characterObject1.get());

keyboardControllerComponent->BindCommandToAction(std::move(moveUpCommand), static_cast<uint32_t>(Action::up));
```

&ensp;
---

### Examples of setting up controls

For the specific controls you can set up a component and bundle which input you want to trigger which action.

<details>
<summary>GamepadControllerComponent.h</summary>

``` C++
#pragma once
#include "ControllerComponent.h"
#include "Controller.h"
#include "Actions.h"

class GamepadControllerComponent final : public dae::ControllerComponent
{
public:
	GamepadControllerComponent(dae::GameObject* gameObject, int controllerIndex)
		:ControllerComponent(gameObject), m_ControllerIndex{ controllerIndex }
	{

	}

private:
	int m_ControllerIndex;

	virtual bool ActionHappened(uint32_t action) override
	{
		auto& controller = dae::Controller::GetInstance();
		Action actionEnum = static_cast<Action>(action);

		switch (actionEnum)
		{
		case Action::up:
			return controller.IsPressed(XINPUT_GAMEPAD_DPAD_UP, m_ControllerIndex);
			break;
		case Action::down:
			return controller.IsPressed(XINPUT_GAMEPAD_DPAD_DOWN, m_ControllerIndex);
			break;
		case Action::left:
			return controller.IsPressed(XINPUT_GAMEPAD_DPAD_LEFT, m_ControllerIndex);
			break;
		case Action::right:
			return controller.IsPressed(XINPUT_GAMEPAD_DPAD_RIGHT, m_ControllerIndex);
			break;
		case Action::push:
			return controller.IsPressed(XINPUT_GAMEPAD_B, m_ControllerIndex);
			break;
		}

		return false;
	}
};
```
</details>

<details>
<summary>KeyboardControllerComponent.h</summary>

``` C++
#pragma once
#include "ControllerComponent.h"
#include "InputManager.h"
#include "Actions.h"

class KeyboardControllerComponent final : public dae::ControllerComponent
{
public:
	KeyboardControllerComponent(dae::GameObject* gameObject)
		:ControllerComponent(gameObject)
	{

	}

private:
	virtual bool ActionHappened(uint32_t action) override
	{
		auto& inputManager = dae::InputManager::GetInstance();
		if (&inputManager == nullptr) return false;
		Action actionEnum = static_cast<Action>(action);

		switch (actionEnum)
		{
		case Action::up:
			return inputManager.IsKeyDown(SDL_SCANCODE_W);
			break;
		case Action::down:
			return inputManager.IsKeyDown(SDL_SCANCODE_S);
			break;
		case Action::left:
			return inputManager.IsKeyDown(SDL_SCANCODE_A);
			break;
		case Action::right:
			return inputManager.IsKeyDown(SDL_SCANCODE_D);
			break;
		case Action::push:
			return inputManager.IsKeyDown(SDL_SCANCODE_SPACE);
			break;
		}

		return false;
	}
};
```
</details>

&ensp;
---
&ensp;

## Implementation of observer pattern

The observer pattern is very useful where you don't need constant polling to happen. When something happens the subject notifies all of its observers that an event happens. The observers can then execute whatever needs to happen. Events are identified with string hashes at compile time, this way you can add new events without having to modify an enum for example. This system is loosely coupled. UI, sound, animation systems, etc. can all listen to the same events, while gameplay logic remains unaware of their existence.

### Observer.h

In my implementation the observer is a simple class that is meant to be inherited from.

``` C++
#pragma once

namespace dae
{
	struct Event;
	class GameObject;

	class Observer
	{
	public:
		virtual ~Observer() = default;
		virtual void Notify(const Event& event, const GameObject* gameObject) = 0;
	};
}

```

### Subject.h

The subject is a component. Any gameobject can add a subject component that is used to dispatch events to observers.

``` C++
#pragma once
#include "Component.h"
#include "Observer.h"
#include <vector>

namespace dae
{
	class Subject : public Component
	{
	public:
		Subject(GameObject* gameObject)
			:Component(gameObject)
		{

		}

		void NotifyObservers(const Event& event)
		{
			for (auto observer : m_observers)
			{
				observer->Notify(event, m_gameObject);
			}
		}

		void AddObserver(Observer* observer)
		{
			m_observers.emplace_back(observer);
		}

		void RemoveObserver(Observer* observer)
		{
			auto it = std::find(std::begin(m_observers), std::end(m_observers), observer);
			if (it != m_observers.end()) m_observers.erase(it);
		}

		virtual void Start() override{}
		virtual void Update() override{}
		virtual void LateUpdate() override{}
		virtual void Render() const override{}
		virtual void RenderUI() const override{}

	private:
		std::vector<Observer*> m_observers{};
	};
}
```

### Event.h

<details>
<summary>Event.h</summary>

``` C++
#pragma once

namespace dae
{
	template <int length> struct sdbm_hash
	{
		constexpr static unsigned int _calculate(const char* const text, unsigned int& value) 
		{
			const unsigned int character = sdbm_hash<length - 1>::_calculate(text, value);
			value = character + (value << 6) + (value << 16) - value;
			return text[length - 1];
		}

		constexpr static unsigned int calculate(const char* const text) 
		{
			unsigned int value = 0;
			const auto character = _calculate(text, value);
			return character + (value << 6) + (value << 16) - value;
		}
	};

	template <> struct sdbm_hash<1> 
	{
		constexpr static int _calculate(const char* const text, unsigned int&) { return text[0]; }
	};

	template <size_t N> constexpr unsigned int make_sdbm_hash(const char(&text)[N])
	{
		return sdbm_hash<N - 1>::calculate(text);
	};

	struct EventArg {};
	using EventId = unsigned int;
	struct Event {
		const EventId id;
		static const uint8_t MAX_ARGS = 8;
		uint8_t nbArgs;
		EventArg args[MAX_ARGS];
		explicit Event(EventId _id) : id{ _id } {}
	};

}
```
</details>

&ensp;
---

### Example of using the pattern

The healthcomponent emits events whenever it takes damage, and when the object dies.

<details>
<summary>Simplified HealthComponent.h</summary>

``` C++
namespace dae
{
	class HealthComponent : public Component
	{
	public:
		HealthComponent(GameObject* gameObject)
			: Component(gameObject)
		{
		}

		virtual void Start() override
		{
			m_Subject = m_gameObject->GetComponent<Subject>();
		}

		float GetHealth() { return m_Health; }
		void TakeDamage(float damage)
		{
			m_Health -= damage;
			if (m_Subject != nullptr)
			{
				// Notify observers to e.g. update UI, play sound, play animation
				m_Subject->NotifyObservers(Event{ make_sdbm_hash("TookDamage") });
				if (m_Health <= 0.f) m_Subject->NotifyObservers(Event{ make_sdbm_hash("Died") });
			}
		}

	private:
		float m_Health{ 100.f };
		Subject* m_Subject{ nullptr };
	};
}
```
</details>

&ensp;

This UI display component listens to these events and updates the UI accordingly.

<details>
<summary>Simplified HealthDisplayComponent.h</summary>

``` C++
namespace dae
{
	class HealthDisplayComponent : public TextComponent, public Observer
	{
	public:
		HealthDisplayComponent(const std::string& text, std::shared_ptr<Font> font, GameObject* gameObject)
			:TextComponent(text, font, gameObject) {}

		virtual void Notify(const Event& event, const GameObject* gameObject) override
		{
			if (event.id == make_sdbm_hash("TookDamage"))
			{
				SetText("health: " + std::to_string(gameObject->GetComponent<HealthComponent>()->GetHealth()));
			}
		}
	};
}
```
</details>

&ensp;
---
&ensp;

## Queued Sound System And Service Locator

To set up a sound system that is easy to access and swap out I used a service locator and a sound system base class which is meant to be inherited from. The service locator serves as a general point of access for the sound system while ensuring only one instance is active. It keeps a null sound system as standard, applying the Null Object pattern.

### SoundSystem.h

``` C++
#pragma once
#include <string>
namespace dae
{
	using SoundId = unsigned short;
	class SoundSystem
	{
	public:
		virtual ~SoundSystem() = default;
		virtual void AddAudioClip(SoundId id, const std::string& filepath) = 0;
		virtual void Play(const SoundId id, const float volume) = 0;
	};
}
```

### ServiceLocator.h

``` C++
#pragma once
#include <memory>
#include <cassert>
#include "NullSoundSystem.h"
namespace dae
{
	class SoundSystem;
	class ServiceLocator final
	{
		static std::unique_ptr<SoundSystem> _ss_instance;
	public:
		static SoundSystem& get_sound_system() 
		{
			return *_ss_instance; 
		}

		static void register_sound_system(std::unique_ptr<SoundSystem>&& ss)
		{
			if (ss != nullptr) _ss_instance = std::move(ss);
			else
			{
				// Unregister it
				_ss_instance.reset();
				_ss_instance = std::make_unique<NullSoundSystem>();
			}
		}
	};
}
```

### ServiceLocator.cpp
``` C++
namespace dae
{
std::unique_ptr<SoundSystem> ServiceLocator::_ss_instance = std::make_unique<NullSoundSystem>();
}
```

### QueuedSoundSystem.h

The queued sound system uses threading to keep the game's framerate from dropping when playing a sound by keeping the playing of sound out of the hot codepath. The cpp file included holds the private implementation that can be swapped out easily, it is intentionally held in a cpp file to hide the specifics.

``` C++
#pragma once
#include "QueuedSoundSystemImplementation.cpp"
namespace dae
{
	class QueuedSoundSystem : public SoundSystem
	{
	public:
		QueuedSoundSystem()
		{
			m_pQueuedSoundSystemImpl = std::make_unique<QueuedSoundSystemImplementation>();
		}

		void AddAudioClip(SoundId id, const std::string& filepath) override
		{
			m_pQueuedSoundSystemImpl->AddAudioClip(id, filepath);
		}

		void Play(const SoundId id, const float volume) override
		{
			m_pQueuedSoundSystemImpl->Play(id, volume);
		}

	private:
		std::unique_ptr<QueuedSoundSystemImplementation> m_pQueuedSoundSystemImpl;

	};
}
```

<details>
<summary>QueuedSoundSystemImplementation.cpp</summary>

``` C++
#pragma once
#include "SoundSystem.h"
#include <unordered_map>
#include <memory>
#include "AudioClip.h"
#include <queue>
#include <thread>
#include <mutex>
namespace dae
{
	struct AudioPlayRequest
	{
		SoundId id;
		float volume;
	};

	class QueuedSoundSystemImplementation : public SoundSystem
	{
	public:
		QueuedSoundSystemImplementation()
		{
			m_AudioThread = std::jthread{ &QueuedSoundSystemImplementation::CycleQueue, this };
			m_StopCycle.store(false);
		}

		~QueuedSoundSystemImplementation()
		{
            // Ensures no dangling work continues after destruction
			m_StopCycle.store(true);
			if (m_AudioThread.joinable())
				m_AudioThread.join();
		}

		void AddAudioClip(SoundId id, const std::string& filepath) override
		{
			m_AudioClips[id] = std::make_unique<AudioClip>(filepath);
		}

		void Play(const SoundId id, const float volume) override
		{
			std::unique_lock<std::mutex> lock(m_QueueMutex);
			m_PlayQueue.push(AudioPlayRequest{ id, volume });
			lock.unlock();
		}

	private:
		void CycleQueue()
		{
			while (!m_StopCycle.load())
			{
				if (!m_PlayQueue.empty())
				{
					std::queue<AudioPlayRequest> localQueue;
					{
						std::unique_lock<std::mutex> lock(m_QueueMutex);
						std::swap(localQueue, m_PlayQueue); // Steal all commands at once
						lock.unlock();
					}

					while (!localQueue.empty())
					{
						const AudioPlayRequest& apr = localQueue.front();

						//play sound
						auto it = m_AudioClips.find(apr.id);
						if (it == m_AudioClips.end()) continue;

						auto& audioclip = it->second;
						if (!audioclip->IsLoaded()) audioclip->Load();

						audioclip->SetVolume(apr.volume);
						audioclip->Play();

						localQueue.pop();
					}
				}
			}
		}

		std::unordered_map<SoundId, std::unique_ptr<AudioClip>> m_AudioClips;
		std::queue<AudioPlayRequest> m_PlayQueue;
		std::mutex m_QueueMutex;
		std::jthread m_AudioThread;
		std::atomic<bool> m_StopCycle;
	};
}
```

</details>

<details>
<summary>NullSoundSystem.h</summary>

``` C++
#pragma once
#include "SoundSystem.h"
namespace dae
{
	class NullSoundSystem : public SoundSystem
	{
		void Play(const SoundId, const float) override {};
		void AddAudioClip(SoundId, const std::string&) override {};
	};
}
```

</details>

<details>
<summary>LoggingSoundSystem.h</summary>

This is used in debug to track if sound clips are actually being added and played. Applying the Decorator pattern.

``` C++
#pragma once
#include "SoundSystem.h"
#include <memory>
#include <iostream>
namespace dae
{
	class LoggingSoundSystem : public SoundSystem
	{
		std::unique_ptr<SoundSystem> _real_ss;
	public:
		LoggingSoundSystem(std::unique_ptr<SoundSystem>&& ss)
			: _real_ss(std::move(ss)) {}
		virtual ~LoggingSoundSystem() = default;

		void AddAudioClip(SoundId id, const std::string& filepath) override
		{
			_real_ss->AddAudioClip(id, filepath);
			std::cout << "added audio clip " << id << " from path " << filepath << std::endl;
		}

		void Play(const SoundId id, const float volume) override
		{
			_real_ss->Play(id, volume);
			std::cout << "playing " << id << " at volume " << volume << std::endl;
		}
	};
}
```

</details>

&ensp;
---

### Usage of service locator and sound system

<details>
<summary>Registering</summary>

``` C++
#if _DEBUG
	ServiceLocator::register_sound_system(std::make_unique<LoggingSoundSystem>(std::make_unique<QueuedSoundSystem>()));
#else
	ServiceLocator::register_sound_system(std::make_unique<QueuedSoundSystem>());
#endif
```
</details>

<details>
<summary>Using</summary>

``` C++
// Getting the system
auto& ss = ServiceLocator::get_sound_system();
// Adding audio clips
ss.AddAudioClip(0, "../Data/CreditSound.wav");
// Playing audio clips
ss.Play(0, 100);
```
</details>
