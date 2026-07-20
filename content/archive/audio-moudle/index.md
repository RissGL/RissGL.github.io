---
title: "基于 ScriptableObject的简单音频模块设计"
date: 2026-07-08
draft: false
---

## 一、AudioSource 管理痛点

在 Unity 中播放音频的硬性前提是——必须有一个挂载了 `AudioSource` 组件的 GameObject。这个前提带来了两个常见的处理方式，各有各的问题：

**方案 A：每个需要播放音频的对象自己挂 AudioSource。** 场景中散落大量 AudioSource 组件，管理混乱。一个角色身上可能挂十几个 AudioSource 来应对不同音效，资源浪费不说，音量、音调、3D 空间混合这些参数每次都要手动设置，容易出错。

**方案 B：写一个 AudioManager 单例统一播放。** 集中管理干净了，但所有播放音频的地方都变成 `AudioManager.Instance.Play(clip)`——调用方要么持有 AudioClip 引用，要么通过字符串/枚举去查表，前者又回到散落资源的问题，后者耦合了全局管理器。

我希望找到一个方式，既保持集中管理的整洁，又能让调用方用最少的代码播放音频。

## 二、让音频"自己播放自己"

核心思路很简单：**用 ScriptableObject 封装 AudioClip 与播放参数**。调用方拿到的是一个 `AudioEvent` 资产，调用 `Play()` 即可，不需要知道 AudioSource 从哪来、怎么配置。

```csharp
public abstract class AudioEvent : ScriptableObject
{
    public abstract void Play();
    public abstract void Play(Vector3 position);
}
```

抽象基类只定义两个接口——2D 播放和 3D 定点播放。具体音频的行为由子类实现。这样做的好处是：

- **资产化**：每个音效是一个 SO 文件，策划可以在 Inspector 里直接调参数
- **类型内聚**：音量范围、音调变化、随机化逻辑都封装在子类里，调用方不关心
- **一键播放**：脚本里 `mySound.Play()` 一行完事

## 三、AudioSource 对象池

ScriptableObject 解决了"配置"的问题，但还需要一个地方提供 AudioSource 实例。这里用一个单例 + 对象池来管理：

```csharp
public class AudioSourceManager : PersistentMonoSingleton<AudioSourceManager>
{
    [SerializeField] private AudioSource audioSourcePrefab;

    private int audioSourceCount = 15;
    private Stack<AudioSource> audioSources = new Stack<AudioSource>();

    protected override void Awake()
    {
        base.Awake();
        if (Instance == this)
        {
            for (int i = 0; i < audioSourceCount; i++)
            {
                AudioSource audioSource = Instantiate(audioSourcePrefab, transform);
                audioSource.gameObject.SetActive(false);
                audioSources.Push(audioSource);
            }
        }
    }

    public AudioSource GetAudioSource()
    {
        AudioSource audioSource;
        if (audioSources.Count > 0)
            audioSource = audioSources.Pop();
        else
            audioSource = Instantiate(audioSourcePrefab, transform);

        audioSource.gameObject.SetActive(true);
        return audioSource;
    }

    public void ReturnAudioSource(AudioSource audioSource, float delay)
    {
        StartCoroutine(DelayReturnAudioSource(audioSource, delay));
    }

    private IEnumerator DelayReturnAudioSource(AudioSource audioSource, float delay)
    {
        yield return new WaitForSeconds(delay);
        audioSource.gameObject.SetActive(false);
        audioSources.Push(audioSource);
    }
}
```

两点说明：

**预初始化 15 个 AudioSource**：覆盖绝大多数场景的音效并发数。超出时自动 Instantiate 扩容，不会漏播。

**按音频时长延迟回收**：播放后根据 `clip.length` 自动归还池子，调用方只需要 `Get` → `Play`，不用自己管理生命周期。`StartCoroutine` 挂载在单例上，场景切换不受影响。

## 四、具体实现示例

`SimpleAudioEvent` 是最常用的子类，支持随机音调/音量变化和 2D/3D 切换：

```csharp
[CreateAssetMenu(menuName = "BaseSoundEffExampleSO")]
public class SimpleAudioEventExample: AudioEvent
{
    [SerializeField] private AudioClip[] clips;
    [SerializeField] private float baseVolume = 1f;
    [SerializeField] private float basePitch = 1f;

    public override void Play()
    {
        if (clips.Length == 0) return;
        AudioSource source = AudioSourceManager.Instance.GetAudioSource();

        source.spatialBlend = 0f;
        source.clip = clips[Random.Range(0, clips.Length)];
        source.volume = baseVolume + Random.Range(-0.1f, 0.1f);
        source.pitch = basePitch + Random.Range(-0.1f, 0.1f);

        source.Play();
        AudioSourceManager.Instance.ReturnAudioSource(source, source.clip.length);
    }

    public override void Play(Vector3 position)
    {
        if (clips.Length == 0) return;
        AudioSource source = AudioSourceManager.Instance.GetAudioSource();

        source.transform.position = position;
        source.clip = clips[Random.Range(0, clips.Length)];
        source.volume = baseVolume + Random.Range(-0.1f, 0.1f);
        source.pitch = basePitch + Random.Range(-0.1f, 0.1f);
        source.spatialBlend = 1f;
        source.minDistance = 3f;
        source.maxDistance = 100f;

        source.Play();
        AudioSourceManager.Instance.ReturnAudioSource(source, source.clip.length);
    }
}
```

使用时只需在 Unity 中右键 `Create → BaseSoundEffSO` 创建资产，拖入 AudioClip、调好参数，然后在代码中：

```csharp
[SerializeField] private SimpleAudioEvent footstepSound;

footstepSound.Play();                    // 2D 播放
footstepSound.Play(transform.position);  // 3D 定点播放
```

不需要引用 AudioManager，不需要知道 AudioSource 的存在。

## 五、存在的问题与改进方向

这个模块的优点是用起来简单，但痛点也很明显：

**配置繁琐。** 每个音效都要手动创建 SO 资产，一个中型项目动辄上百个音效，逐个创建和维护十分低效。理想方案是写一个编辑器工具，指定音频文件夹后批量生成 SO 资产，类似 Addressables 的工作流。

## 六、总结

这个模块的核心思想一句话概括：**让音频资产成为一个自包含的对象，拥有播放自己的能力**。ScriptableObject 作为配置载体，AudioSourceManager 作为播放能力的提供方，两者通过抽象基类解耦。虽然还有很多可改进的地方，但在日常开发中，拖一个 SO 进来然后 `.Play()` 的体验比到处写 `GetComponent<AudioSource>` 或 `AudioManager.Instance.Play("footstep")` 舒服得多。

目前只是一个简单的实现，未来将会做个完整的音频模块。
