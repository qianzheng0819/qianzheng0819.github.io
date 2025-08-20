sequenceDiagram
    autonumber
    participant C as Client(调用方)
    participant L as ActivityInitListener
    participant T as ActivityTracker<T>
    participant A as BaseActivity(例如 LauncherActivity)

    Note over C,L: 构造时：new ActivityInitListener(onInitListener, tracker)

    C->>L: register(reason)
    L->>L: mIsRegistered = true
    L->>T: registerCallback(L, reason)

    alt Activity 已经创建
        T-->>L: init(activity=A, alreadyOnHome=?)
        L->>L: mIsRegistered ? (true)
        L->>L: handleInit(A, alreadyOnHome)
        L->>C: onInitListener.test(A, alreadyOnHome) -> boolean
        alt 返回 true
            Note over L: 保持注册，后续重建仍会回调
        else 返回 false
            Note over L: 该次回调链结束（Tracker侧不再继续）
        end
    else Activity 尚未创建
        Note over T,A: 未来某时刻 Activity 创建完成
        A-->>T: onActivityReady
        T-->>L: init(activity=A, alreadyOnHome=?)
        L->>L: mIsRegistered ? (true)
        L->>L: handleInit(A, alreadyOnHome)
        L->>C: onInitListener.test(A, alreadyOnHome) -> boolean
        alt 返回 true
            Note over L: 继续保持注册（应对重建）
        else 返回 false
            Note over L: 该次回调链结束
        end
    end

    %% 解绑流程
    C->>L: unregister(reason)
    L->>T: unregisterCallback(L, reason)
    L->>L: mIsRegistered = false; mOnInitListener = null

    %% 后续即使 Activity 准备就绪也不会触发
    T-->>L: init(activity, alreadyOnHome)
    L->>L: mIsRegistered ? (false) => return false
