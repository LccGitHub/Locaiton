#GPS 初始化流程#

##LocationManager--modem流程分析##

###1.1 LocationManagerService.java###
该service类启动后，<code>void systemRuning()</code>会被调用,主要是加载provider已经启动导航
<pre><code>
void systemRunning() {
...
    loadProvidersLocked();
    updateProvidersLocked();
}
</code></pre>

该<code>loadProvidersLocked</code>函数主要是加载提供位置信息的provider
<pre><code>
void loadProvidersLocked() {  
    PassiveProvider passiveProvider = new PassiveProvider(this);
    addProviderLocked(passiveProvider);
    mEnabledProviders.add(passiveProvider.getName());
    mPassiveProvider = passiveProvider;


	GnssLocationProvider gnssProvider = new GnssLocationProvider(this);
	addProviderLocked(gnssProvider);


	LocationProviderProxy networkProvider = new LocationProviderProxy(
    ....
}
</code></pre>

该<code>updateProvidersLocked</code>函数主要决定provider的是否enable
<pre><code>
void loadProvidersLocked() {  
    for (int i = mProviders.size() - 1; i >= 0; i--) {
    LocationProviderInterface p = mProviders.get(i);
    boolean isEnabled = p.isEnabled();
    String name = p.getName();
	boolean shouldBeEnabled = isAllowedByCurrentUserSettingsLocked(name);
	if (isEnabled && !shouldBeEnabled) {
		updateProviderListenersLocked(name, false);
	} else if (!isEnabled && shouldBeEnabled) {
		updateProviderListenersLocked(name, true);
	}
}
</code></pre>
该<code>updateProviderListenersLocked</code>主要是调用相关provider的enable/disable函数
<pre><code>
void updateProviderListenersLocked(String provider, boolean enabled) {  
	...
    if (enabled) {
    	p.enable();
	} else {
		p.disable();
	}
}
</code></pre>


###1.2 以GnssLocationProvider为例分析provider的实现###

该<code>enable()</code>主要是发送消息给自己的线程处理
<pre><code>
void enable() {  
	...
    if (enabled) {
    	； /*have enabled*/
	} else {
		sendMessage(ENABLE, 1, null);
	}
}
</code></pre>

该线程收到上面的消息,进行内部的处理
<pre><code>
public void handleMessage(Message msg) {
	int message = msg.what;
	switch (message) {
		case ENABLE:
			if (msg.arg1 == 1) {
				handleEnable();
			} else {
					handleDisable();
			 }
			 break;
		case ....
	}
}
</code></pre>

<code>handleEnable</code>调用JNI层的代码
<pre><code>
public void handleEnable() {
	boolean enabled = native_init();
	if (enabled) {
		mSupportsXtra = native_supports_xtra();
			if (mSuplServerHost != null) {
				native_set_agps_server(AGPS_TYPE_SUPL, mSuplServerHost, mSuplServerPort);
			} 
			if (mC2KServerHost != null) {
				native_set_agps_server(AGPS_TYPE_C2K, mC2KServerHost, mC2KServerPort);
			 }
			mGnssMeasurementsProvider.onGpsEnabledChanged();
			GnssNavigationMessageProvider.onGpsEnabledChanged();
	}
	else {
		 mEnabled = false;
	}	
}
</code></pre>


###1.3 JNI代码com_android_server_location_GnssLocationProvider.cpp实现###

<code>native_init()</code>其实就是<code>android_location_GnssLocationProvider_init()</code>
该函数主要是注册HAL的callback函数
<pre><code>
static jboolean android_location_GnssLocationProvider_init(JNIEnv* env, jobject obj)
{
    // this must be set before calling into the HAL library
    if (!mCallbacksObj)
        mCallbacksObj = env->NewGlobalRef(obj);

    // fail if the main interface fails to initialize
    if (!sGpsInterface || sGpsInterface->init(&sGpsCallbacks) != 0)
        return JNI_FALSE;

    // if XTRA initialization fails we will disable it by sGpsXtraInterface to NULL,
    // but continue to allow the rest of the GPS interface to work.
    if (sGpsXtraInterface && sGpsXtraInterface->init(&sGpsXtraCallbacks) != 0)
        sGpsXtraInterface = NULL;
    if (sAGpsInterface)
        sAGpsInterface->init(&sAGpsCallbacks);
    if (sGpsNiInterface)
        sGpsNiInterface->init(&sGpsNiCallbacks);
    if (sAGpsRilInterface)
        sAGpsRilInterface->init(&sAGpsRilCallbacks);
    if (sGpsGeofencingInterface)
        sGpsGeofencingInterface->init(&sGpsGeofenceCallbacks);

    return JNI_TRUE;
}
</code></pre>