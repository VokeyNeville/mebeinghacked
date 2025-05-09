import { __extends } from "tslib";
/**
* PostManager.ts
* @author Abhilash Panwar (abpanwar); Hector Hernandez (hectorh); Nev Wylie (newylie)
* @copyright Microsoft 2018-2020
*/
import dynamicProto from "@microsoft/dynamicproto-js";
import { BaseTelemetryPlugin, EventsDiscardedReason, _throwInternal, addPageHideEventListener, addPageShowEventListener, addPageUnloadEventListener, arrForEach, createUniqueNamespace, doPerf, getWindow, isChromium, isNumber, isValueAssigned, mergeEvtNamespace, objDefineAccessors, objForEachKey, optimizeObject, removePageHideEventListener, removePageShowEventListener, removePageUnloadEventListener, setProcessTelemetryTimings } from "@microsoft/1ds-core-js";
import { BE_PROFILE, NRT_PROFILE, RT_PROFILE } from "./DataModels";
import { EventBatch } from "./EventBatch";
import { HttpManager } from "./HttpManager";
import { STR_MSA_DEVICE_TICKET, STR_TRACE, STR_USER } from "./InternalConstants";
import { retryPolicyGetMillisToBackoffForRetry } from "./RetryPolicy";
import { createTimeoutWrapper } from "./TimeoutOverrideWrapper";
var FlushCheckTimer = 0.250; // This needs to be in seconds, so this is 250ms
var MaxNumberEventPerBatch = 500;
var EventsDroppedAtOneTime = 20;
var MaxSendAttempts = 6;
var MaxSyncUnloadSendAttempts = 2; // Assuming 2 based on beforeunload and unload
var MaxBackoffCount = 4;
var MaxConnections = 2;
var MaxRequestRetriesBeforeBackoff = 1;
var strEventsDiscarded = "eventsDiscarded";
var strOverrideInstrumentationKey = "overrideInstrumentationKey";
var strMaxEventRetryAttempts = "maxEventRetryAttempts";
var strMaxUnloadEventRetryAttempts = "maxUnloadEventRetryAttempts";
var strAddUnloadCb = "addUnloadCb";
/**
 * Class that manages adding events to inbound queues and batching of events
 * into requests.
 */
var PostChannel = /** @class */ (function (_super) {
    __extends(PostChannel, _super);
    function PostChannel() {
        var _this = _super.call(this) || this;
        _this.identifier = "PostChannel";
        _this.priority = 1011;
        _this.version = '3.2.15';
        var _config;
        var _isTeardownCalled = false;
        var _flushCallbackQueue = [];
        var _flushCallbackTimerId = null;
        var _paused = false;
        var _immediateQueueSize = 0;
        var _immediateQueueSizeLimit = 500;
        var _queueSize = 0;
        var _queueSizeLimit = 10000;
        var _profiles = {};
        var _currentProfile = RT_PROFILE;
        var _scheduledTimerId = null;
        var _immediateTimerId = null;
        var _currentBackoffCount = 0;
        var _timerCount = 0;
        var _xhrOverride;
        var _httpManager;
        var _batchQueues = {};
        var _autoFlushEventsLimit;
        // either MaxBatchSize * (1+ Max Connections) or _queueLimit / 6 (where 3 latency Queues [normal, realtime, cost deferred] * 2 [allow half full -- allow for retry])
        var _autoFlushBatchLimit;
        var _delayedBatchSendLatency = -1;
        var _delayedBatchReason;
        var _optimizeObject = true;
        var _isPageUnloadTriggered = false;
        var _maxEventSendAttempts = MaxSendAttempts;
        var _maxUnloadEventSendAttempts = MaxSyncUnloadSendAttempts;
        var _evtNamespace;
        var _timeoutWrapper;
        dynamicProto(PostChannel, _this, function (_self, _base) {
            _initDefaults();
            // Special internal method to allow the DebugPlugin to hook embedded objects
            _self["_getDbgPlgTargets"] = function () {
                return [_httpManager];
            };
            _self.initialize = function (coreConfig, core, extensions) {
                doPerf(core, function () { return "PostChannel:initialize"; }, function () {
                    var extendedCore = core;
                    _base.initialize(coreConfig, core, extensions);
                    try {
                        var hasAddUnloadCb = !!core[strAddUnloadCb];
                        _evtNamespace = mergeEvtNamespace(createUniqueNamespace(_self.identifier), core.evtNamespace && core.evtNamespace());
                        var ctx = _self._getTelCtx();
                        coreConfig.extensionConfig[_self.identifier] = coreConfig.extensionConfig[_self.identifier] || {};
                        _config = ctx.getExtCfg(_self.identifier);
                        _timeoutWrapper = createTimeoutWrapper(_config.setTimeoutOverride, _config.clearTimeoutOverride);
                        // Only try and use the optimizeObject() if this appears to be a chromium based browser and it has not been explicitly disabled
                        _optimizeObject = !_config.disableOptimizeObj && isChromium();
                        _hookWParam(extendedCore);
                        if (_config.eventsLimitInMem > 0) {
                            _queueSizeLimit = _config.eventsLimitInMem;
                        }
                        if (_config.immediateEventLimit > 0) {
                            _immediateQueueSizeLimit = _config.immediateEventLimit;
                        }
                        if (_config.autoFlushEventsLimit > 0) {
                            _autoFlushEventsLimit = _config.autoFlushEventsLimit;
                        }
                        if (isNumber(_config[strMaxEventRetryAttempts])) {
                            _maxEventSendAttempts = _config[strMaxEventRetryAttempts];
                        }
                        if (isNumber(_config[strMaxUnloadEventRetryAttempts])) {
                            _maxUnloadEventSendAttempts = _config[strMaxUnloadEventRetryAttempts];
                        }
                        _setAutoLimits();
                        if (_config.httpXHROverride && _config.httpXHROverride.sendPOST) {
                            _xhrOverride = _config.httpXHROverride;
                        }
                        if (isValueAssigned(coreConfig.anonCookieName)) {
                            _httpManager.addQueryStringParameter("anoncknm", coreConfig.anonCookieName);
                        }
                        _httpManager.sendHook = _config.payloadPreprocessor;
                        _httpManager.sendListener = _config.payloadListener;
                        // Override endpointUrl if provided in Post config
                        var endpointUrl = _config.overrideEndpointUrl ? _config.overrideEndpointUrl : coreConfig.endpointUrl;
                        _self._notificationManager = core.getNotifyMgr();
                        _httpManager.initialize(endpointUrl, _self.core, _self, _xhrOverride, _config);
                        var excludePageUnloadEvents = coreConfig.disablePageUnloadEvents || [];
                        // When running in Web browsers try to send all telemetry if page is unloaded
                        addPageUnloadEventListener(_handleUnloadEvents, excludePageUnloadEvents, _evtNamespace);
                        addPageHideEventListener(_handleUnloadEvents, excludePageUnloadEvents, _evtNamespace);
                        addPageShowEventListener(_handleShowEvents, coreConfig.disablePageShowEvents, _evtNamespace);
                    }
                    catch (e) {
                        // resetting the initialized state because of failure
                        _self.setInitialized(false);
                        throw e;
                    }
                }, function () { return ({ coreConfig: coreConfig, core: core, extensions: extensions }); });
            };
            _self.processTelemetry = function (ev, itemCtx) {
                setProcessTelemetryTimings(ev, _self.identifier);
                itemCtx = _self._getTelCtx(itemCtx);
                // Get the channel instance from the current request/instance
                var channelConfig = itemCtx.getExtCfg(_self.identifier);
                // DisableTelemetry was defined in the config provided during initialization
                var disableTelemetry = !!_config.disableTelemetry;
                if (channelConfig) {
                    // DisableTelemetry is defined in the config for this request/instance
                    disableTelemetry = disableTelemetry || !!channelConfig.disableTelemetry;
                }
                var event = ev;
                if (!disableTelemetry && !_isTeardownCalled) {
                    // Override iKey if provided in Post config if provided for during initialization
                    if (_config[strOverrideInstrumentationKey]) {
                        event.iKey = _config[strOverrideInstrumentationKey];
                    }
                    // Override iKey if provided in Post config if provided for this instance
                    if (channelConfig && channelConfig[strOverrideInstrumentationKey]) {
                        event.iKey = channelConfig[strOverrideInstrumentationKey];
                    }
                    _addEventToQueues(event, true);
                    if (_isPageUnloadTriggered) {
                        // Unload event has been received so we need to try and flush new events
                        _releaseAllQueues(2 /* EventSendType.SendBeacon */, 2 /* SendRequestReason.Unload */);
                    }
                    else {
                        _scheduleTimer();
                    }
                }
                _self.processNext(event, itemCtx);
            };
            _self._doTeardown = function (unloadCtx, unloadState) {
                _releaseAllQueues(2 /* EventSendType.SendBeacon */, 2 /* SendRequestReason.Unload */);
                _isTeardownCalled = true;
                _httpManager.teardown();
                removePageUnloadEventListener(null, _evtNamespace);
                removePageHideEventListener(null, _evtNamespace);
                removePageShowEventListener(null, _evtNamespace);
                // Just register to remove all events associated with this namespace
                _initDefaults();
            };
            function _hookWParam(extendedCore) {
                var existingGetWParamMethod = extendedCore.getWParam;
                extendedCore.getWParam = function () {
                    var wparam = 0;
                    if (_config.ignoreMc1Ms0CookieProcessing) {
                        wparam = wparam | 2;
                    }
                    return wparam | existingGetWParamMethod();
                };
            }
            // Moving event handlers out from the initialize closure so that any local variables can be garbage collected
            function _handleUnloadEvents(evt) {
                var theEvt = evt || getWindow().event; // IE 8 does not pass the event
                if (theEvt.type !== "beforeunload") {
                    // Only set the unload trigger if not beforeunload event as beforeunload can be cancelled while the other events can't
                    _isPageUnloadTriggered = true;
                    _httpManager.setUnloading(_isPageUnloadTriggered);
                }
                _releaseAllQueues(2 /* EventSendType.SendBeacon */, 2 /* SendRequestReason.Unload */);
            }
            function _handleShowEvents(evt) {
                // Handle the page becoming visible again
                _isPageUnloadTriggered = false;
                _httpManager.setUnloading(_isPageUnloadTriggered);
            }
            function _addEventToQueues(event, append) {
                // If send attempt field is undefined we should set it to 0.
                if (!event.sendAttempt) {
                    event.sendAttempt = 0;
                }
                // Add default latency
                if (!event.latency) {
                    event.latency = 1 /* EventLatencyValue.Normal */;
                }
                // Remove extra AI properties if present
                if (event.ext && event.ext[STR_TRACE]) {
                    delete (event.ext[STR_TRACE]);
                }
                if (event.ext && event.ext[STR_USER] && event.ext[STR_USER]["id"]) {
                    delete (event.ext[STR_USER]["id"]);
                }
                // v8 performance optimization for iterating over the keys
                if (_optimizeObject) {
                    setProcessTelemetryTimings;
                    event.ext = optimizeObject(event.ext);
                    if (event.baseData) {
                        event.baseData = optimizeObject(event.baseData);
                    }
                    if (event.data) {
                        event.data = optimizeObject(event.data);
                    }
                }
                if (event.sync) {
                    // If the transmission is backed off then do not send synchronous events.
                    // We will convert these events to Real time latency instead.
                    if (_currentBackoffCount || _paused) {
                        event.latency = 3 /* EventLatencyValue.RealTime */;
                        event.sync = false;
                    }
                    else {
                        // Log the event synchronously
                        if (_httpManager) {
                            // v8 performance optimization for iterating over the keys
                            if (_optimizeObject) {
                                event = optimizeObject(event);
                            }
                            _httpManager.sendSynchronousBatch(EventBatch.create(event.iKey, [event]), event.sync === true ? 1 /* EventSendType.Synchronous */ : event.sync, 3 /* SendRequestReason.SyncEvent */);
                            return;
                        }
                    }
                }
                var evtLatency = event.latency;
                var queueSize = _queueSize;
                var queueLimit = _queueSizeLimit;
                if (evtLatency === 4 /* EventLatencyValue.Immediate */) {
                    queueSize = _immediateQueueSize;
                    queueLimit = _immediateQueueSizeLimit;
                }
                var eventDropped = false;
                // Only add the event if the queue isn't full or it's a direct event (which don't add to the queue sizes)
                if (queueSize < queueLimit) {
                    eventDropped = !_addEventToProperQueue(event, append);
                }
                else {
                    var dropLatency = 1 /* EventLatencyValue.Normal */;
                    var dropNumber = EventsDroppedAtOneTime;
                    if (evtLatency === 4 /* EventLatencyValue.Immediate */) {
                        // Only drop other immediate events as they are not technically sharing the general queue
                        dropLatency = 4 /* EventLatencyValue.Immediate */;
                        dropNumber = 1;
                    }
                    // Drop old event from lower or equal latency
                    eventDropped = true;
                    if (_dropEventWithLatencyOrLess(event.iKey, event.latency, dropLatency, dropNumber)) {
                        eventDropped = !_addEventToProperQueue(event, append);
                    }
                }
                if (eventDropped) {
                    // Can't drop events from current queues because the all the slots are taken by queues that are being flushed.
                    _notifyEvents(strEventsDiscarded, [event], EventsDiscardedReason.QueueFull);
                }
            }
            _self.setEventQueueLimits = function (eventLimit, autoFlushLimit) {
                _queueSizeLimit = eventLimit > 0 ? eventLimit : 10000;
                _autoFlushEventsLimit = autoFlushLimit > 0 ? autoFlushLimit : 0;
                _setAutoLimits();
                // We only do this check here as during normal event addition if the queue is > then events start getting dropped
                var doFlush = _queueSize > eventLimit;
                if (!doFlush && _autoFlushBatchLimit > 0) {
                    // Check the auto flush max batch size
                    for (var latency = 1 /* EventLatencyValue.Normal */; !doFlush && latency <= 3 /* EventLatencyValue.RealTime */; latency++) {
                        var batchQueue = _batchQueues[latency];
                        if (batchQueue && batchQueue.batches) {
                            arrForEach(batchQueue.batches, function (theBatch) {
                                if (theBatch && theBatch.count() >= _autoFlushBatchLimit) {
                                    // If any 1 batch is > than the limit then trigger an auto flush
                                    doFlush = true;
                                }
                            });
                        }
                    }
                }
                _performAutoFlush(true, doFlush);
            };
            _self.pause = function () {
                _clearScheduledTimer();
                _paused = true;
                _httpManager.pause();
            };
            _self.resume = function () {
                _paused = false;
                _httpManager.resume();
                _scheduleTimer();
            };
            _self.addResponseHandler = function (responseHandler) {
                _httpManager._responseHandlers.push(responseHandler);
            };
            _self._loadTransmitProfiles = function (profiles) {
                _resetTransmitProfiles();
                objForEachKey(profiles, function (profileName, profileValue) {
                    var profLen = profileValue.length;
                    if (profLen >= 2) {
                        var directValue = (profLen > 2 ? profileValue[2] : 0);
                        profileValue.splice(0, profLen - 2);
                        // Make sure if a higher latency is set to not send then don't send lower latency
                        if (profileValue[1] < 0) {
                            profileValue[0] = -1;
                        }
                        // Make sure each latency is multiple of the latency higher then it. If not a multiple
                        // we round up so that it becomes a multiple.
                        if (profileValue[1] > 0 && profileValue[0] > 0) {
                            var timerMultiplier = profileValue[0] / profileValue[1];
                            profileValue[0] = Math.ceil(timerMultiplier) * profileValue[1];
                        }
                        // Add back the direct profile timeout
                        if (directValue >= 0 && profileValue[1] >= 0 && directValue > profileValue[1]) {
                            // Make sure if it's not disabled (< 0) then make sure it's not larger than RealTime
                            directValue = profileValue[1];
                        }
                        profileValue.push(directValue);
                        _profiles[profileName] = profileValue;
                    }
                });
            };
            _self.flush = function (async, callback, sendReason) {
                if (async === void 0) { async = true; }
                if (!_paused) {
                    sendReason = sendReason || 1 /* SendRequestReason.ManualFlush */;
                    if (async) {
                        if (_flushCallbackTimerId == null) {
                            // Clear the normal schedule timer as we are going to try and flush ASAP
                            _clearScheduledTimer();
                            // Move all queued events to the HttpManager so that we don't discard new events (Auto flush scenario)
                            _queueBatches(1 /* EventLatencyValue.Normal */, 0 /* EventSendType.Batched */, sendReason);
                            _flushCallbackTimerId = _createTimer(function () {
                                _flushCallbackTimerId = null;
                                _flushImpl(callback, sendReason);
                            }, 0);
                        }
                        else {
                            // Even if null (no callback) this will ensure after the flushImpl finishes waiting
                            // for a completely idle connection it will attempt to re-flush any queued events on the next cycle
                            _flushCallbackQueue.push(callback);
                        }
                    }
                    else {
                        // Clear the normal schedule timer as we are going to try and flush ASAP
                        var cleared = _clearScheduledTimer();
                        // Now cause all queued events to be sent synchronously
                        _sendEventsForLatencyAndAbove(1 /* EventLatencyValue.Normal */, 1 /* EventSendType.Synchronous */, sendReason);
                        if (callback !== null && callback !== undefined) {
                            callback();
                        }
                        if (cleared) {
                            // restart the normal event timer if it was cleared
                            _scheduleTimer();
                        }
                    }
                }
            };
            _self.setMsaAuthTicket = function (ticket) {
                _httpManager.addHeader(STR_MSA_DEVICE_TICKET, ticket);
            };
            _self.hasEvents = _hasEvents;
            _self._setTransmitProfile = function (profileName) {
                if (_currentProfile !== profileName && _profiles[profileName] !== undefined) {
                    _clearScheduledTimer();
                    _currentProfile = profileName;
                    _scheduleTimer();
                }
            };
            /**
             * Batch and send events currently in the queue for the given latency.
             * @param latency - Latency for which to send events.
             */
            function _sendEventsForLatencyAndAbove(latency, sendType, sendReason) {
                var queued = _queueBatches(latency, sendType, sendReason);
                // Always trigger the request as while the post channel may not have queued additional events, the httpManager may already have waiting events
                _httpManager.sendQueuedRequests(sendType, sendReason);
                return queued;
            }
            function _hasEvents() {
                return _queueSize > 0;
            }
            /**
             * Try to schedule the timer after which events will be sent. If there are
             * no events to be sent, or there is already a timer scheduled, or the
             * http manager doesn't have any idle connections this method is no-op.
             */
            function _scheduleTimer() {
                // If we had previously attempted to send requests, but the http manager didn't have any idle connections then the requests where delayed
                // so try and requeue then again now
                if (_delayedBatchSendLatency >= 0 && _queueBatches(_delayedBatchSendLatency, 0 /* EventSendType.Batched */, _delayedBatchReason)) {
                    _httpManager.sendQueuedRequests(0 /* EventSendType.Batched */, _delayedBatchReason);
                }
                if (_immediateQueueSize > 0 && !_immediateTimerId && !_paused) {
                    // During initialization _profiles enforce that the direct [2] is less than real time [1] timer value
                    // If the immediateTimeout is disabled the immediate events will be sent with Real Time events
                    var immediateTimeOut = _profiles[_currentProfile][2];
                    if (immediateTimeOut >= 0) {
                        _immediateTimerId = _createTimer(function () {
                            _immediateTimerId = null;
                            // Only try to send direct events
                            _sendEventsForLatencyAndAbove(4 /* EventLatencyValue.Immediate */, 0 /* EventSendType.Batched */, 1 /* SendRequestReason.NormalSchedule */);
                            _scheduleTimer();
                        }, immediateTimeOut);
                    }
                }
                // During initialization the _profiles enforce that the normal [0] is a multiple of the real time [1] timer value
                var timeOut = _profiles[_currentProfile][1];
                if (!_scheduledTimerId && !_flushCallbackTimerId && timeOut >= 0 && !_paused) {
                    if (_hasEvents()) {
                        _scheduledTimerId = _createTimer(function () {
                            _scheduledTimerId = null;
                            _sendEventsForLatencyAndAbove(_timerCount === 0 ? 3 /* EventLatencyValue.RealTime */ : 1 /* EventLatencyValue.Normal */, 0 /* EventSendType.Batched */, 1 /* SendRequestReason.NormalSchedule */);
                            // Increment the count for next cycle
                            _timerCount++;
                            _timerCount %= 2;
                            _scheduleTimer();
                        }, timeOut);
                    }
                    else {
                        _timerCount = 0;
                    }
                }
            }
            _self._backOffTransmission = function () {
                if (_currentBackoffCount < MaxBackoffCount) {
                    _currentBackoffCount++;
                    _clearScheduledTimer();
                    _scheduleTimer();
                }
            };
            _self._clearBackOff = function () {
                if (_currentBackoffCount) {
                    _currentBackoffCount = 0;
                    _clearScheduledTimer();
                    _scheduleTimer();
                }
            };
            function _initDefaults() {
                _config = null;
                _isTeardownCalled = false;
                _flushCallbackQueue = [];
                _flushCallbackTimerId = null;
                _paused = false;
                _immediateQueueSize = 0;
                _immediateQueueSizeLimit = 500;
                _queueSize = 0;
                _queueSizeLimit = 10000;
                _profiles = {};
                _currentProfile = RT_PROFILE;
                _scheduledTimerId = null;
                _immediateTimerId = null;
                _currentBackoffCount = 0;
                _timerCount = 0;
                _xhrOverride = null;
                _batchQueues = {};
                _autoFlushEventsLimit = undefined;
                // either MaxBatchSize * (1+ Max Connections) or _queueLimit / 6 (where 3 latency Queues [normal, realtime, cost deferred] * 2 [allow half full -- allow for retry])
                _autoFlushBatchLimit = 0;
                _delayedBatchSendLatency = -1;
                _delayedBatchReason = null;
                _optimizeObject = true;
                _isPageUnloadTriggered = false;
                _maxEventSendAttempts = MaxSendAttempts;
                _maxUnloadEventSendAttempts = MaxSyncUnloadSendAttempts;
                _evtNamespace = null;
                _timeoutWrapper = createTimeoutWrapper();
                _httpManager = new HttpManager(MaxNumberEventPerBatch, MaxConnections, MaxRequestRetriesBeforeBackoff, {
                    requeue: _requeueEvents,
                    send: _sendingEvent,
                    sent: _eventsSentEvent,
                    drop: _eventsDropped,
                    rspFail: _eventsResponseFail,
                    oth: _otherEvent
                }, _timeoutWrapper);
                _initializeProfiles();
                _clearQueues();
                _setAutoLimits();
            }
            function _createTimer(theTimerFunc, timeOut) {
                // If the transmission is backed off make the timer at least 1 sec to allow for back off.
                if (timeOut === 0 && _currentBackoffCount) {
                    timeOut = 1;
                }
                var timerMultiplier = 1000;
                if (_currentBackoffCount) {
                    timerMultiplier = retryPolicyGetMillisToBackoffForRetry(_currentBackoffCount - 1);
                }
                return _timeoutWrapper.set(theTimerFunc, timeOut * timerMultiplier);
            }
            function _clearScheduledTimer() {
                if (_scheduledTimerId !== null) {
                    _timeoutWrapper.clear(_scheduledTimerId);
                    _scheduledTimerId = null;
                    _timerCount = 0;
                    return true;
                }
                return false;
            }
            // Try to send all queued events using beacons if available
            function _releaseAllQueues(sendType, sendReason) {
                _clearScheduledTimer();
                // Cancel all flush callbacks
                if (_flushCallbackTimerId) {
                    _timeoutWrapper.clear(_flushCallbackTimerId);
                    _flushCallbackTimerId = null;
                }
                if (!_paused) {
                    // Queue all the remaining requests to be sent. The requests will be sent using HTML5 Beacons if they are available.
                    _sendEventsForLatencyAndAbove(1 /* EventLatencyValue.Normal */, sendType, sendReason);
                }
            }
            /**
             * Add empty queues for all latencies in the inbound queues map. This is called
             * when Transmission Manager is being flushed. This ensures that new events added
             * after flush are stored separately till we flush the current events.
             */
            function _clearQueues() {
                _batchQueues[4 /* EventLatencyValue.Immediate */] = {
                    batches: [],
                    iKeyMap: {}
                };
                _batchQueues[3 /* EventLatencyValue.RealTime */] = {
                    batches: [],
                    iKeyMap: {}
                };
                _batchQueues[2 /* EventLatencyValue.CostDeferred */] = {
                    batches: [],
                    iKeyMap: {}
                };
                _batchQueues[1 /* EventLatencyValue.Normal */] = {
                    batches: [],
                    iKeyMap: {}
                };
            }
            function _getEventBatch(iKey, latency, create) {
                var batchQueue = _batchQueues[latency];
                if (!batchQueue) {
                    latency = 1 /* EventLatencyValue.Normal */;
                    batchQueue = _batchQueues[latency];
                }
                var eventBatch = batchQueue.iKeyMap[iKey];
                if (!eventBatch && create) {
                    eventBatch = EventBatch.create(iKey);
                    batchQueue.batches.push(eventBatch);
                    batchQueue.iKeyMap[iKey] = eventBatch;
                }
                return eventBatch;
            }
            function _performAutoFlush(isAsync, doFlush) {
                // Only perform the auto flush check if the httpManager has an idle connection and we are not in a backoff situation
                if (_httpManager.canSendRequest() && !_currentBackoffCount) {
                    if (_autoFlushEventsLimit > 0 && _queueSize > _autoFlushEventsLimit) {
                        // Force flushing
                        doFlush = true;
                    }
                    if (doFlush && _flushCallbackTimerId == null) {
                        // Auto flush the queue
                        _self.flush(isAsync, null, 20 /* SendRequestReason.MaxQueuedEvents */);
                    }
                }
            }
            function _addEventToProperQueue(event, append) {
                // v8 performance optimization for iterating over the keys
                if (_optimizeObject) {
                    event = optimizeObject(event);
                }
                var latency = event.latency;
                var eventBatch = _getEventBatch(event.iKey, latency, true);
                if (eventBatch.addEvent(event)) {
                    if (latency !== 4 /* EventLatencyValue.Immediate */) {
                        _queueSize++;
                        // Check for auto flushing based on total events in the queue, but not for requeued or retry events
                        if (append && event.sendAttempt === 0) {
                            // Force the flushing of the batch if the batch (specific iKey / latency combination) reaches it's auto flush limit
                            _performAutoFlush(!event.sync, _autoFlushBatchLimit > 0 && eventBatch.count() >= _autoFlushBatchLimit);
                        }
                    }
                    else {
                        // Direct events don't need auto flushing as they are scheduled (by default) for immediate delivery
                        _immediateQueueSize++;
                    }
                    return true;
                }
                return false;
            }
            function _dropEventWithLatencyOrLess(iKey, latency, currentLatency, dropNumber) {
                while (currentLatency <= latency) {
                    var eventBatch = _getEventBatch(iKey, latency, true);
                    if (eventBatch && eventBatch.count() > 0) {
                        // Dropped oldest events from lowest possible latency
                        var droppedEvents = eventBatch.split(0, dropNumber);
                        var droppedCount = droppedEvents.count();
                        if (droppedCount > 0) {
                            if (currentLatency === 4 /* EventLatencyValue.Immediate */) {
                                _immediateQueueSize -= droppedCount;
                            }
                            else {
                                _queueSize -= droppedCount;
                            }
                            _notifyBatchEvents(strEventsDiscarded, [droppedEvents], EventsDiscardedReason.QueueFull);
                            return true;
                        }
                    }
                    currentLatency++;
                }
                // Unable to drop any events -- lets just make sure the queue counts are correct to avoid exhaustion
                _resetQueueCounts();
                return false;
            }
            /**
             * Internal helper to reset the queue counts, used as a backstop to avoid future queue exhaustion errors
             * that might occur because of counting issues.
             */
            function _resetQueueCounts() {
                var immediateQueue = 0;
                var normalQueue = 0;
                var _loop_1 = function (latency) {
                    var batchQueue = _batchQueues[latency];
                    if (batchQueue && batchQueue.batches) {
                        arrForEach(batchQueue.batches, function (theBatch) {
                            if (latency === 4 /* EventLatencyValue.Immediate */) {
                                immediateQueue += theBatch.count();
                            }
                            else {
                                normalQueue += theBatch.count();
                            }
                        });
                    }
                };
                for (var latency = 1 /* EventLatencyValue.Normal */; latency <= 4 /* EventLatencyValue.Immediate */; latency++) {
                    _loop_1(latency);
                }
                _queueSize = normalQueue;
                _immediateQueueSize = immediateQueue;
            }
            function _queueBatches(latency, sendType, sendReason) {
                var eventsQueued = false;
                var isAsync = sendType === 0 /* EventSendType.Batched */;
                // Only queue batches (to the HttpManager) if this is a sync request or the httpManager has an idle connection
                // Thus keeping the events within the PostChannel until the HttpManager has a connection available
                // This is so we can drop "old" events if the queue is getting full because we can't successfully send events
                if (!isAsync || _httpManager.canSendRequest()) {
                    doPerf(_self.core, function () { return "PostChannel._queueBatches"; }, function () {
                        var droppedEvents = [];
                        var latencyToProcess = 4 /* EventLatencyValue.Immediate */;
                        while (latencyToProcess >= latency) {
                            var batchQueue = _batchQueues[latencyToProcess];
                            if (batchQueue && batchQueue.batches && batchQueue.batches.length > 0) {
                                arrForEach(batchQueue.batches, function (theBatch) {
                                    // Add the batch to the http manager to send the requests
                                    if (!_httpManager.addBatch(theBatch)) {
                                        // The events from this iKey are being dropped (killed)
                                        droppedEvents = droppedEvents.concat(theBatch.events());
                                    }
                                    else {
                                        eventsQueued = eventsQueued || (theBatch && theBatch.count() > 0);
                                    }
                                    if (latencyToProcess === 4 /* EventLatencyValue.Immediate */) {
                                        _immediateQueueSize -= theBatch.count();
                                    }
                                    else {
                                        _queueSize -= theBatch.count();
                                    }
                                });
                                // Remove all batches from this Queue
                                batchQueue.batches = [];
                                batchQueue.iKeyMap = {};
                            }
                            latencyToProcess--;
                        }
                        if (droppedEvents.length > 0) {
                            _notifyEvents(strEventsDiscarded, droppedEvents, EventsDiscardedReason.KillSwitch);
                        }
                        if (eventsQueued && _delayedBatchSendLatency >= latency) {
                            // We have queued events at the same level as the delayed values so clear the setting
                            _delayedBatchSendLatency = -1;
                            _delayedBatchReason = 0 /* SendRequestReason.Undefined */;
                        }
                    }, function () { return ({ latency: latency, sendType: sendType, sendReason: sendReason }); }, !isAsync);
                }
                else {
                    // remember the min latency so that we can re-trigger later
                    _delayedBatchSendLatency = _delayedBatchSendLatency >= 0 ? Math.min(_delayedBatchSendLatency, latency) : latency;
                    _delayedBatchReason = Math.max(_delayedBatchReason, sendReason);
                }
                return eventsQueued;
            }
            /**
             * This is the callback method is called as part of the manual flushing process.
             * @param callback
             * @param sendReason
             */
            function _flushImpl(callback, sendReason) {
                // Add any additional queued events and cause all queued events to be sent asynchronously
                _sendEventsForLatencyAndAbove(1 /* EventLatencyValue.Normal */, 0 /* EventSendType.Batched */, sendReason);
                // All events (should) have been queue -- lets just make sure the queue counts are correct to avoid queue exhaustion (previous bug #9685112)
                _resetQueueCounts();
                _waitForIdleManager(function () {
                    // Only called AFTER the httpManager does not have any outstanding requests
                    if (callback) {
                        callback();
                    }
                    if (_flushCallbackQueue.length > 0) {
                        _flushCallbackTimerId = _createTimer(function () {
                            _flushCallbackTimerId = null;
                            _flushImpl(_flushCallbackQueue.shift(), sendReason);
                        }, 0);
                    }
                    else {
                        // No more flush requests
                        _flushCallbackTimerId = null;
                        // Restart the normal timer schedule
                        _scheduleTimer();
                    }
                });
            }
            function _waitForIdleManager(callback) {
                if (_httpManager.isCompletelyIdle()) {
                    callback();
                }
                else {
                    _flushCallbackTimerId = _createTimer(function () {
                        _flushCallbackTimerId = null;
                        _waitForIdleManager(callback);
                    }, FlushCheckTimer);
                }
            }
            /**
             * Resets the transmit profiles to the default profiles of Real Time, Near Real Time
             * and Best Effort. This removes all the custom profiles that were loaded.
             */
            function _resetTransmitProfiles() {
                _clearScheduledTimer();
                _initializeProfiles();
                _currentProfile = RT_PROFILE;
                _scheduleTimer();
            }
            function _initializeProfiles() {
                _profiles = {};
                _profiles[RT_PROFILE] = [2, 1, 0];
                _profiles[NRT_PROFILE] = [6, 3, 0];
                _profiles[BE_PROFILE] = [18, 9, 0];
            }
            /**
             * The notification handler for requeue events
             * @ignore
             */
            function _requeueEvents(batches, reason) {
                var droppedEvents = [];
                var maxSendAttempts = _maxEventSendAttempts;
                if (_isPageUnloadTriggered) {
                    // If a page unlaod has been triggered reduce the number of times we try to "retry"
                    maxSendAttempts = _maxUnloadEventSendAttempts;
                }
                arrForEach(batches, function (theBatch) {
                    if (theBatch && theBatch.count() > 0) {
                        arrForEach(theBatch.events(), function (theEvent) {
                            if (theEvent) {
                                // Check if the request being added back is for a sync event in which case mark it no longer a sync event
                                if (theEvent.sync) {
                                    theEvent.latency = 4 /* EventLatencyValue.Immediate */;
                                    theEvent.sync = false;
                                }
                                if (theEvent.sendAttempt < maxSendAttempts) {
                                    // Reset the event timings
                                    setProcessTelemetryTimings(theEvent, _self.identifier);
                                    _addEventToQueues(theEvent, false);
                                }
                                else {
                                    droppedEvents.push(theEvent);
                                }
                            }
                        });
                    }
                });
                if (droppedEvents.length > 0) {
                    _notifyEvents(strEventsDiscarded, droppedEvents, EventsDiscardedReason.NonRetryableStatus);
                }
                if (_isPageUnloadTriggered) {
                    // Unload event has been received so we need to try and flush new events
                    _releaseAllQueues(2 /* EventSendType.SendBeacon */, 2 /* SendRequestReason.Unload */);
                }
            }
            function _callNotification(evtName, theArgs) {
                var manager = (_self._notificationManager || {});
                var notifyFunc = manager[evtName];
                if (notifyFunc) {
                    try {
                        notifyFunc.apply(manager, theArgs);
                    }
                    catch (e) {
                        _throwInternal(_self.diagLog(), 1 /* eLoggingSeverity.CRITICAL */, 74 /* _eInternalMessageId.NotificationException */, evtName + " notification failed: " + e);
                    }
                }
            }
            function _notifyEvents(evtName, theEvents) {
                var extraArgs = [];
                for (var _i = 2; _i < arguments.length; _i++) {
                    extraArgs[_i - 2] = arguments[_i];
                }
                if (theEvents && theEvents.length > 0) {
                    _callNotification(evtName, [theEvents].concat(extraArgs));
                }
            }
            function _notifyBatchEvents(evtName, batches) {
                var extraArgs = [];
                for (var _i = 2; _i < arguments.length; _i++) {
                    extraArgs[_i - 2] = arguments[_i];
                }
                if (batches && batches.length > 0) {
                    arrForEach(batches, function (theBatch) {
                        if (theBatch && theBatch.count() > 0) {
                            _callNotification(evtName, [theBatch.events()].concat(extraArgs));
                        }
                    });
                }
            }
            /**
             * The notification handler for when batches are about to be sent
             * @ignore
             */
            function _sendingEvent(batches, reason, isSyncRequest) {
                if (batches && batches.length > 0) {
                    _callNotification("eventsSendRequest", [(reason >= 1000 /* EventBatchNotificationReason.SendingUndefined */ && reason <= 1999 /* EventBatchNotificationReason.SendingEventMax */ ?
                            reason - 1000 /* EventBatchNotificationReason.SendingUndefined */ :
                            0 /* SendRequestReason.Undefined */), isSyncRequest !== true]);
                }
            }
            /**
             * This event represents that a batch of events have been successfully sent and a response received
             * @param batches The notification handler for when the batches have been successfully sent
             * @param reason For this event the reason will always be EventBatchNotificationReason.Complete
             */
            function _eventsSentEvent(batches, reason) {
                _notifyBatchEvents("eventsSent", batches, reason);
                // Try and schedule the processing timer if we have events
                _scheduleTimer();
            }
            function _eventsDropped(batches, reason) {
                _notifyBatchEvents(strEventsDiscarded, batches, (reason >= 8000 /* EventBatchNotificationReason.EventsDropped */ && reason <= 8999 /* EventBatchNotificationReason.EventsDroppedMax */ ?
                    reason - 8000 /* EventBatchNotificationReason.EventsDropped */ :
                    EventsDiscardedReason.Unknown));
            }
            function _eventsResponseFail(batches) {
                _notifyBatchEvents(strEventsDiscarded, batches, EventsDiscardedReason.NonRetryableStatus);
                // Try and schedule the processing timer if we have events
                _scheduleTimer();
            }
            function _otherEvent(batches, reason) {
                _notifyBatchEvents(strEventsDiscarded, batches, EventsDiscardedReason.Unknown);
                // Try and schedule the processing timer if we have events
                _scheduleTimer();
            }
            function _setAutoLimits() {
                if (!_config || !_config.disableAutoBatchFlushLimit) {
                    _autoFlushBatchLimit = Math.max(MaxNumberEventPerBatch * (MaxConnections + 1), _queueSizeLimit / 6);
                }
                else {
                    _autoFlushBatchLimit = 0;
                }
            }
            // Provided for backward compatibility they are not "expected" to be in current use but they are public
            objDefineAccessors(_self, "_setTimeoutOverride", function () { return _timeoutWrapper.set; }, function (value) {
                // Recreate the timeout wrapper
                _timeoutWrapper = createTimeoutWrapper(value, _timeoutWrapper.clear);
            });
            objDefineAccessors(_self, "_clearTimeoutOverride", function () { return _timeoutWrapper.clear; }, function (value) {
                // Recreate the timeout wrapper
                _timeoutWrapper = createTimeoutWrapper(_timeoutWrapper.set, value);
            });
        });
        return _this;
    }
    /**
     * Start the queue manager to batch and send events via post.
     * @param config - The core configuration.
     */
    PostChannel.prototype.initialize = function (coreConfig, core, extensions) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Add an event to the appropriate inbound queue based on its latency.
     * @param ev - The event to be added to the queue.
     * @param itemCtx - This is the context for the current request, ITelemetryPlugin instances
     * can optionally use this to access the current core instance or define / pass additional information
     * to later plugins (vs appending items to the telemetry item)
     */
    PostChannel.prototype.processTelemetry = function (ev, itemCtx) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Sets the event queue limits at runtime (after initialization), if the number of queued events is greater than the
     * eventLimit or autoFlushLimit then a flush() operation will be scheduled.
     * @param eventLimit The number of events that can be kept in memory before the SDK starts to drop events. If the value passed is less than or
     * equal to zero the value will be reset to the default (10,000).
     * @param autoFlushLimit When defined, once this number of events has been queued the system perform a flush() to send the queued events
     * without waiting for the normal schedule timers. Passing undefined, null or a value less than or equal to zero will disable the auto flush.
     */
    PostChannel.prototype.setEventQueueLimits = function (eventLimit, autoFlushLimit) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Pause the transmission of any requests
     */
    PostChannel.prototype.pause = function () {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Resumes transmission of events.
     */
    PostChannel.prototype.resume = function () {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
    * Add handler to be executed with request response text.
    */
    PostChannel.prototype.addResponseHandler = function (responseHanlder) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Flush to send data immediately; channel should default to sending data asynchronously
     * @param async - send data asynchronously when true
     * @param callback - if specified, notify caller when send is complete
     */
    PostChannel.prototype.flush = function (async, callback, sendReason) {
        if (async === void 0) { async = true; }
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Set AuthMsaDeviceTicket header
     * @param ticket - Ticket value.
     */
    PostChannel.prototype.setMsaAuthTicket = function (ticket) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Check if there are any events waiting to be scheduled for sending.
     * @returns True if there are events, false otherwise.
     */
    PostChannel.prototype.hasEvents = function () {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
        return false;
    };
    /**
     * Load custom transmission profiles. Each profile should have timers for real time, and normal and can
     * optionally specify the immediate latency time in ms (defaults to 0 when not defined). Each profile should
     * make sure that a each normal latency timer is a multiple of the real-time latency and the immediate
     * is smaller than the real-time.
     * Setting the timer value to -1 means that the events for that latency will not be scheduled to be sent.
     * Note that once a latency has been set to not send, all latencies below it will also not be sent. The
     * timers should be in the form of [normal, high, [immediate]].
     * e.g Custom:
     * [10,5] - Sets the normal latency time to 10 seconds and real-time to 5 seconds; Immediate will default to 0ms
     * [10,5,1] - Sets the normal latency time to 10 seconds and real-time to 5 seconds; Immediate will default to 1ms
     * [10,5,0] - Sets the normal latency time to 10 seconds and real-time to 5 seconds; Immediate will default to 0ms
     * [10,5,-1] - Sets the normal latency time to 10 seconds and real-time to 5 seconds; Immediate events will not be
     * scheduled on their own and but they will be included with real-time or normal events as the first events in a batch.
     * This also removes any previously loaded custom profiles.
     * @param profiles - A dictionary containing the transmit profiles.
     */
    PostChannel.prototype._loadTransmitProfiles = function (profiles) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Set the transmit profile to be used. This will change the transmission timers
     * based on the transmit profile.
     * @param profileName - The name of the transmit profile to be used.
     */
    PostChannel.prototype._setTransmitProfile = function (profileName) {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Backs off transmission. This exponentially increases all the timers.
     */
    PostChannel.prototype._backOffTransmission = function () {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    /**
     * Clears backoff for transmission.
     */
    PostChannel.prototype._clearBackOff = function () {
        // @DynamicProtoStub - DO NOT add any code as this will be removed during packaging
    };
    return PostChannel;
}(BaseTelemetryPlugin));
export default PostChannel;
//# sourceMappingURL=PostChannel.js.map
