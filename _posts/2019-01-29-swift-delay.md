---
layout: post
title: "swift Throttle, Debounce"
description: "액션을 일정시간동안 후에 실행"
date: 2019-01-29
tags: [throttle, debounce, search, delay]
writer: pikachu987
category: ios
comments: true
share: true
---

## Debounce과 Throttle

이벤트를 일정시간 이후에 보여주게 합니다.

예를들어 UITextField에서 텍스트를 입력하고 입력된 텍스트를 통신하게 되면 큰 비용이 일어나게 됩니다.

이 문제를 해결하려면 텍스트를 입력한 다음 일정시간동안 다음 텍스트를 입력받지 못하면 통신하고 일정시간 안에 텍스트를 입력받으면 다시 다음 일정시간을 기다리게 할수 있고 큰 비용이 발생하지 않습니다.


### Debounce

이벤트가 일어날때마다 항상 취소후 다시 시간 딜레이를 줍니다.

```swift
class CustomTextField: UITextField {

    deinit {
        self.removeTarget(self, action: #selector(self.editingChanged(_:)), for: .editingChanged)
    }

    private var workItem: DispatchWorkItem?
    private var delay: Double = 0
    private var callback: ((String?) -> Void)? = nil

    func debounce(delay: Double, callback: @escaping ((String?) -> Void)) {
        self.delay = delay
        self.callback = callback
        DispatchQueue.main.async {
            self.callback?(self.text)
        }
        self.addTarget(self, action: #selector(self.editingChanged(_:)), for: .editingChanged)
    }

    @objc private func editingChanged(_ sender: UITextField) {
      self.workItem?.cancel()
      let workItem = DispatchWorkItem(block: { [weak self] in
          self?.callback?(sender.text)
      })
      self.workItem = workItem
      DispatchQueue.main.asyncAfter(deadline: .now() + self.delay, execute: workItem)
    }
}
```

```swift
let textField = CustomTextField()
textField.debounce(delay: 0.3) { (text) in
    print(text)
}
```

이제 0.3초 후에 텍스트를 받게 됩니다. 0.3초가 흐르기 전에 텍스트를 입력하면 이벤트 호출이 되지 않고 다시 0.3초를 기다리게 됩니다.

![debounce]({{ site.url }}{{ site.baseurl }}/images/2019/delay/debounce.png)

### Throttle

지정한 시간 간격으로 마지막 이벤트만 호출됩니다.

```swift
class CustomTextField: UITextField {

    deinit {
        self.removeTarget(self, action: #selector(self.editingChanged(_:)), for: .editingChanged)
    }

    private var workItem: DispatchWorkItem?
    private var delay: Double = 0
    private var callback: ((String?) -> Void)? = nil

    func throttle(delay: Double, callback: @escaping ((String?) -> Void)) {
        self.delay = delay
        self.callback = callback
        DispatchQueue.main.async {
            self.callback?(self.text)
        }
        self.addTarget(self, action: #selector(self.editingChanged(_:)), for: .editingChanged)
    }

    @objc private func editingChanged(_ sender: UITextField) {
        if self.workItem == nil {
            let text = sender.text
            self.callback?(text)
            let workItem = DispatchWorkItem(block: { [weak self] in
                self?.workItem?.cancel()
                self?.workItem = nil
                if text != sender.text {
                    self?.callback?(sender.text)
                }
            })
            self.workItem = workItem
            DispatchQueue.main.asyncAfter(deadline: .now() + self.delay, execute: workItem)
        }
    }
}
```

```swift
let textField = CustomTextField()
textField.throttle(delay: 0.3) { (text) in
    print(text)
}
```

0.3초 동안 일어난 이벤트중 마지막 이벤트가 호출되고 다시 0.3초를 기다립니다.

해당 예제는 RxSwift에서 throttle 메서드의 동작과 비슷하게 행동합니다.
> 텍스트 입력시 바로 이벤트가 호출되고 특정 시간동안 이벤트가 발생하지 않으면 호출되지 않고 이벤트가 발생하면 마지막 이벤트만 호출됩니다.

![throttle]({{ site.url }}{{ site.baseurl }}/images/2019/delay/throttle.png)
