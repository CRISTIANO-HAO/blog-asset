# piwik源码解析

## addTracker

添加一个tracker，`addTracker` :

```javascript
if (!window.Piwik.getAsyncTrackers().length) {
  // 我们只在piwikAsyncInit()中尚未创建其他异步跟踪器时创建初始跟踪器
  if (hasPaqConfiguration()) {
    // 如果通过_paq有配置，我们只创建一个初始跟踪器。 除此以外
    // Piwik.getAsyncTrackers()将返回未配置的跟踪器
    window.Piwik.addTracker();
  } else {
    _paq = {push: function (args) {
      // needed to write it this way for jslint
      var consoleType = typeof console;
      if (consoleType !== 'undefined' && console && console.error) {
        console.error('_paq.push() was used', args);
      }
    }};
  }
}
```

## createFirstTracker

通过`createFirstTracker` 创建第一个tracker：

```javascript
addTracker: function (piwikUrl, siteId) {
  var tracker;
  if (!asyncTrackers.length) {
    tracker = createFirstTracker(piwikUrl, siteId);
  } else {
    tracker = asyncTrackers[0].addTracker(piwikUrl, siteId);
  }
  return tracker;
},
```

实例化第一个tracker：

```javascript
//创建第一个跟踪器
function createFirstTracker(piwikUrl, siteId)
{
  var tracker = new Tracker(piwikUrl, siteId);
  asyncTrackers.push(tracker);

  _paq = applyMethodsInOrder(_paq, applyFirst);

  // apply the queue of actions
  for (iterator = 0; iterator < _paq.length; iterator++) {
    if (_paq[iterator]) {
      apply(_paq[iterator]);
    }
  }

  // 用代理对象替换初始化数组
  _paq = new TrackerProxy();

  return tracker;
}
```

通过`apply` 方法执行paq数组中内置的节点后移除：

```javascript
function applyMethodsInOrder(paq, methodsToApply)
{
  var appliedMethods = {};
  var index, iterator;

  for (index = 0; index < methodsToApply.length; index++) {
    var methodNameToApply = methodsToApply[index];
    appliedMethods[methodNameToApply] = 1;

    for (iterator = 0; iterator < paq.length; iterator++) {
      if (paq[iterator] && paq[iterator][0]) {
        var methodName = paq[iterator][0];

        if (methodNameToApply === methodName) {
          apply(paq[iterator]);
          delete paq[iterator];

          if (appliedMethods[methodName] > 1
              && methodName !== "addTracker") {
            logConsoleError('The method ' + methodName + ' is r');
          }
          appliedMethods[methodName]++;
        }
      }
    }
  }

  return paq;
}
```

## apply

`apply` 方法实际上是代理执行`asyncTrackers` 数组上的方法：

```javascript
// [ 'methodName', optional_parameters ]
function apply() {
  var i, j, f, parameterArray, trackerCall;

  for (i = 0; i < arguments.length; i += 1) {
    trackerCall = null;
    if (arguments[i] && arguments[i].slice) {
      trackerCall = arguments[i].slice();
    }
    parameterArray = arguments[i];
    f = parameterArray.shift();

    var fParts, context;

    var isStaticPluginCall = isString(f) && f.indexOf("::") > 0;
    if (isStaticPluginCall) {
      // ...
    } else {
      for (j = 0; j < asyncTrackers.length; j++) {
        if (isString(f)) {
          context = asyncTrackers[j];

          var isPluginTrackerCall = f.indexOf(".") > 0;

          // ...

          if (context[f]) {
            context[f].apply(context, parameterArray);
          } else {
            // ...
        } else {
          f.apply(asyncTrackers[j], parameterArray);
        }
      }
    }
  }
}
```

`_paq` 数组被代理，`push` 方法实际上执行的是上面的 `apply` 方法：

```javascript
// 用代理对象替换初始化数组
_paq = new TrackerProxy();
function TrackerProxy() {
  return {
    push: apply
  };
}
```

## trackCallback

`trackCallback` 包裹`track` 逻辑，`callback` 是具体的逻辑实现。监控 `document.visibilityState === 'prerender'` 状态时，在`visibilitychange` 事件里执行回调：

```javascript
function trackCallback(callback) {
  var isPreRendered,
      i,
      // Chrome 13, IE10, FF10
      prefixes = ["", "webkit", "ms", "moz"],
      prefix;

  if (!configCountPreRendered) {
    for (i = 0; i < prefixes.length; i++) {
      prefix = prefixes[i];

      // does this browser support the page visibility API?
      if (
        Object.prototype.hasOwnProperty.call(
          documentAlias,
          prefixPropertyName(prefix, "hidden")
        )
      ) {
        // if pre-rendered, then defer callback until page visibility changes
        if (
          documentAlias[prefixPropertyName(prefix, "visibilityState")] ===
          "prerender"
        ) {
          isPreRendered = true;
        }
        break;
      }
    }
  }

  if (isPreRendered) {
    // note: the event name doesn't follow the same naming convention as vendor properties
    addEventListener(
      documentAlias,
      prefix + "visibilitychange",
      function ready() {
        documentAlias.removeEventListener(
          prefix + "visibilitychange",
          ready,
          false
        );
        callback();
      }
    );

    return;
  }

  // configCountPreRendered === true || isPreRendered === false
  callback();
}
```



## getRequest

此方法生成请求URL。

### getValuesFromVisitorIdCookie

从cookie中获取相关数据：

```javascript
function getValuesFromVisitorIdCookie() {
  var cookieVisitorIdValue = loadVisitorIdCookie(),
      newVisitor = cookieVisitorIdValue[0],
      uuid = cookieVisitorIdValue[1],
      createTs = cookieVisitorIdValue[2],
      visitCount = cookieVisitorIdValue[3],
      currentVisitTs = cookieVisitorIdValue[4],
      lastVisitTs = cookieVisitorIdValue[5];

  // case migrating from pre-1.5 cookies
  if (!isDefined(cookieVisitorIdValue[6])) {
    cookieVisitorIdValue[6] = "";
  }

  var lastEcommerceOrderTs = cookieVisitorIdValue[6];

  return {
    newVisitor: newVisitor,
    uuid: uuid,
    createTs: createTs,
    visitCount: visitCount,
    currentVisitTs: currentVisitTs,
    lastVisitTs: lastVisitTs,
    lastEcommerceOrderTs: lastEcommerceOrderTs
  };
}
```

session每一次请求都会刷新过期时间，默认30min过期，每过期一次算一次全新访问，`visitCount++`：

```javascript
if (!cookieSessionValue) {
  // cookie 'ses' was not found: we consider this the start of a 'session'

  // here we make sure that if 'ses' cookie is deleted few times within the visit
  // and so this code path is triggered many times for one visit,
  // we only increase visitCount once per Visit window (default 30min)
  var visitDuration = configSessionCookieTimeout / 1000;
  if (!cookieVisitorIdValues.lastVisitTs ||
      nowTs - cookieVisitorIdValues.lastVisitTs > visitDuration
     ) {
    cookieVisitorIdValues.visitCount++;
    cookieVisitorIdValues.lastVisitTs =
      cookieVisitorIdValues.currentVisitTs;
  }
  // ...
}
```



## sendRequest

发起请求，记录数据。

```javascript
function sendRequest(request, delay, callback) {
  if (!configHasConsent) {
    consentRequestsQueue.push(request);
    return;
  }
  if (!configDoNotTrack && request) {
    if (configConsentRequired && configHasConsent) {
      // send a consent=1 when explicit consent is given for the apache logs
      request += "&consent=1";
    }

    makeSureThereIsAGapAfterFirstTrackingRequestToPreventMultipleVisitorCreation(
      function () {
        if (configRequestMethod === "POST" || String(request).length > 2000) {
          sendXmlHttpRequest(request, callback);
        } else {
          getImage(request, callback);
        }

        setExpireDateTime(delay);
      }
    );
  }
  if (!heartBeatSetUp) {
    setUpHeartBeat(); // setup window events too, but only once
  } else {
    heartBeatUp();
  }
}
```

### makeSureThereIsAGapAfterFirstTrackingRequestToPreventMultipleVisitorCreation

避免频繁请求，两次请求间隔不能小于50ms：

```javascript
function makeSureThereIsAGapAfterFirstTrackingRequestToPreventMultipleVisitorCreation(
    callback
  ) {
      var now = new Date();
      var timeNow = now.getTime();

      lastTrackerRequestTime = timeNow;

      if (
        timeNextTrackingRequestCanBeExecutedImmediately &&
        timeNow < timeNextTrackingRequestCanBeExecutedImmediately
      ) {
        // we are in the time frame shortly after the first request. we have to delay this request a bit to make sure
        // a visitor has been created meanwhile.

        var timeToWait =
            timeNextTrackingRequestCanBeExecutedImmediately - timeNow;

        setTimeout(callback, timeToWait);
        setExpireDateTime(timeToWait + 50); // set timeout is not necessarily executed at timeToWait so delay a bit more
        timeNextTrackingRequestCanBeExecutedImmediately += 50; // delay next tracking request by further 50ms to next execute them at same time

        return;
      }

      if (timeNextTrackingRequestCanBeExecutedImmediately === false) {
        // it is the first request, we want to execute this one directly and delay all the next one(s) within a delay.
        // All requests after this delay can be executed as usual again
        var delayInMs = 800;
        timeNextTrackingRequestCanBeExecutedImmediately = timeNow + delayInMs;
      }

      callback();
    }
```



### getImage

当URL长度小于2000时，默认使用get请求，通过创建一个image标签发起请求：

```javascript
function getImage(request, callback) {
  // make sure to actually load an image so callback gets invoked
  request = request.replace("send_image=0", "send_image=1");

  var image = new Image(1, 1);
  image.onload = function () {
    iterator = 0; // To avoid JSLint warning of empty block
    if (typeof callback === "function") {
      callback();
    }
  };
  image.src =
    configTrackerUrl +
    (configTrackerUrl.indexOf("?") < 0 ? "?" : "&") +
    request;
}
```

### sendXmlHttpRequest

post 请求或者URL过大时，使用ajax发起请求。

```javascript
function sendXmlHttpRequest(request, callback, fallbackToGet) {
    if (!isDefined(fallbackToGet) || null === fallbackToGet) {
      fallbackToGet = true;
    }

    if (isPageUnloading && sendPostRequestViaSendBeacon(request)) {
      return;
    }

    setTimeout(function () {
      // we execute it with a little delay in case the unload event occurred just after sending this request
      // this is to avoid the following behaviour: Eg on form submit a tracking request is sent via POST
      // in this method. Then a few ms later the browser wants to navigate to the new page and the unload
      // event occurrs and the browser cancels the just triggered POST request. This causes or fallback
      // method to be triggered and we execute the same request again (either as fallbackGet or sendBeacon).
      // The problem is that we do not know whether the inital POST request was already fully transferred
      // to the server or not when the onreadystatechange callback is executed and we might execute the
      // same request a second time. To avoid this, we delay the actual execution of this POST request just
      // by 50ms which gives it usually enough time to detect the unload event in most cases.

      if (isPageUnloading && sendPostRequestViaSendBeacon(request)) {
        return;
      }
      var sentViaBeacon;

      try {
        // we use the progid Microsoft.XMLHTTP because
        // IE5.5 included MSXML 2.5; the progid MSXML2.XMLHTTP
        // is pinned to MSXML2.XMLHTTP.3.0
        var xhr = windowAlias.XMLHttpRequest ? new windowAlias.XMLHttpRequest() : windowAlias.ActiveXObject ? new ActiveXObject("Microsoft.XMLHTTP") : null;

        xhr.open("POST", configTrackerUrl, true);

        // fallback on error
        xhr.onreadystatechange = function () {
          if (
            this.readyState === 4 &&
            !(this.status >= 200 && this.status < 300)
          ) {
            var sentViaBeacon =
              isPageUnloading && sendPostRequestViaSendBeacon(request);

            if (!sentViaBeacon && fallbackToGet) {
              getImage(request, callback);
            }
          } else {
            if (this.readyState === 4 && typeof callback === "function") {
              callback();
            }
          }
        };

        xhr.setRequestHeader("Content-Type", configRequestContentType);

        xhr.send(request);
      } catch (e) {
        sentViaBeacon =
          isPageUnloading && sendPostRequestViaSendBeacon(request);
        if (!sentViaBeacon && fallbackToGet) {
          getImage(request, callback);
        }
      }
    }, 50);
  }
```

### sendPostRequestViaSendBeacon

`navigator.sendBeacon(url, data);` 用户代理通常会忽略在 `unload ` 事件处理器中产生的异步 `XMLHttpRequest`。

为了解决这个问题， 统计和诊断代码通常要在 `unload` 或者 `beforeunload ` 事件处理器中发起一个同步 `XMLHttpRequest` 来发送数据。同步的 `XMLHttpRequest` 迫使用户代理延迟卸载文档，并使得下一个导航出现的更晚。下一个页面对于这种较差的载入表现无能为力。

有一些技术被用来保证数据的发送。

- 其中一种是通过在卸载事件处理器中创建一个图片元素并设置它的 src 属性的方法来延迟卸载以保证数据的发送。因为绝大多数用户代理会延迟卸载以保证图片的载入，所以数据可以在卸载事件中发送。
- 另一种技术是通过创建一个几秒钟的 no-op 循环来延迟卸载并向服务器发送数据。

使用 **`sendBeacon() `**方法会使用户代理在有机会时异步地向服务器发送数据，同时不会延迟页面的卸载或影响下一导航的载入性能。这就解决了提交分析数据时的所有的问题：数据可靠，传输异步并且不会影响下一页面的加载。

```javascript
function sendPostRequestViaSendBeacon(request) {
  var supportsSendBeacon =
      "object" === typeof navigatorAlias &&
      "function" === typeof navigatorAlias.sendBeacon &&
      "function" === typeof Blob;

  if (!supportsSendBeacon) {
    return false;
  }

  var headers = {
    type: "application/x-www-form-urlencoded; charset=UTF-8"
  };
  var success = false;

  try {
    var blob = new Blob([request], headers);
    success = navigatorAlias.sendBeacon(configTrackerUrl, blob);
    // returns true if the user agent is able to successfully queue the data for transfer,
    // Otherwise it returns false and we need to try the regular way
  } catch (e) {
    return false;
  }

  return success;
}
```

