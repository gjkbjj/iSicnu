# 登录与课表源码节选

以下 Kotlin 代码摘自 iSicnu Android 主应用的登录、密码存储和课表读取路径。代码块保留原逻辑，供对照阅读。它们是源码节选，依赖原项目的部分类型，无法单独构建 APK。

## 教务账号登录

手动登录在 App 内的校方认证页完成。开启「快速登录 · Beta」后，App 只在学校的 HTTPS 统一认证登录页填入已保存的密码。部分学校旧教务跳转仍使用 HTTP，教务密码提交限于学校的 HTTPS 认证页。

`AuthPolicy.kt` 的校方认证主机与凭据页面校验

```kotlin
const val AUTH_HOST = "authserver.sicnu.edu.cn"
// AuthPolicy.allows 的校方认证分支
AUTH_HOST -> url.isHttps && url.port == 443

fun canFillCredentials(url: String): Boolean = url.toHttpUrlOrNull()?.let {
    allows(it) && it.host == AUTH_HOST && it.encodedPath == "/authserver/login"
} == true
```

`BetaCredentials.kt` 的浏览器快速登录页面与表单检查

```javascript
if (location.protocol !== 'https:' || location.hostname !== 'authserver.sicnu.edu.cn' ||
    location.port && location.port !== '443' || location.pathname !== '/authserver/login') return 'unsafe';
// …查找并验证表单控件…
const action = new URL(form.action || location.href, location.href);
if (action.origin !== location.origin || action.pathname !== '/authserver/login' ||
    form.method.toLowerCase() !== 'post') return 'unsafe';
```

`SessionRecovery.kt` 的后台认证表单节选。客户端关闭自动重定向，每次发送带密码的请求前重新校验目标地址。

```kotlin
fields.remove("passwordText")
fields["username"] = local.account
fields["password"] = encrypt(local.password, salt)
fields["_eventId"] = "submit"
fields["cllt"] = "userNameLogin"
fields["dllt"] = "generalLogin"
body = FormBody.Builder().apply { fields.forEach { (key, value) -> add(key, value) } }.build()
```

```kotlin
val request = Request.Builder().url(url).header("User-Agent", AuthPolicy.USER_AGENT)
body?.let { request.post(it).header("Origin", "https://${AuthPolicy.AUTH_HOST}") }
if (body != null && !AuthPolicy.canFillCredentials(url.toString())) throw VerificationRequired(VerificationReason.UNSUPPORTED_PAGE)
```

## 教务密码如何保存在手机上

只有用户同意开启「快速登录 · Beta」，且学号与当前登录账号一致，App 才保存教务密码。保存时先用 Android Keystore 管理的 AES-256 密钥加密，再写入 App 的本机私有目录。关闭该账号的快速登录或退出账号可以删除保存的密码。

`BetaCredentials.kt` 的同意校验

```kotlin
fun validate(currentAccount: String, consent: Boolean) {
    require(consent && account == currentAccount && studentType(currentAccount) != StudentType.UNKNOWN)
    require(password.isNotEmpty())
}
```

`BetaCredentials.kt` 的存取入口

```kotlin
private val file = SecureFile(context, "beta-credentials.enc", preserveOnReadFailure = true)
```

```kotlin
fun save(credentials: BetaCredentials, currentAccount: String, consent: Boolean) = synchronized(lock) {
    credentials.validate(currentAccount, consent)
    file.write(CredentialBookCodec.encode(readCredentials().filterNot { it.account == currentAccount } + credentials))
}

fun clear(number: String? = null) = synchronized(lock) {
    if (number == null) file.clear()
    else file.write(CredentialBookCodec.encode(readCredentials().filterNot { it.account == number }))
}
```

`SecureFile.kt` 的本机加密写入

```kotlin
private val file = AtomicFile(File(context.noBackupFilesDir, name))

fun write(json: JSONObject) = synchronized(lock) {
    val bytes = CacheCipher.encrypt(json.toString().toByteArray(Charsets.UTF_8))
    val stream = file.startWrite()
    try { stream.write(bytes); file.finishWrite(stream) }
    catch (error: Exception) { file.failWrite(stream); throw error }
}
```

`CacheCipher.kt` 的密钥管理与 AES-GCM 加密

```kotlin
private fun key(): SecretKey {
    val alias = "isicnu.session.v1"
    val store = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }
    (store.getKey(alias, null) as? SecretKey)?.let { return it }
    return KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore").apply {
        init(KeyGenParameterSpec.Builder(alias, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM).setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .setKeySize(256).build())
    }.generateKey()
}

@Synchronized fun encrypt(bytes: ByteArray, aad: ByteArray? = null): ByteArray = encryptWithKey(bytes, key(), aad)
```

```kotlin
internal fun encryptWithKey(bytes: ByteArray, key: SecretKey, aad: ByteArray? = null): ByteArray {
    val cipher = Cipher.getInstance("AES/GCM/NoPadding").apply { init(Cipher.ENCRYPT_MODE, key) }
    aad?.let(cipher::updateAAD)
    return cipher.iv + cipher.doFinal(bytes)
}
```

## 课表如何读取

客户端按学生类型，使用已建立的校方会话从本科课表服务或研究生教务门户读取课表。课表读取使用会话，而无需重新传递教务密码。本机课表缓存按账号隔离并加密。

`AcademicClient.kt` 的课表读取入口

```kotlin
private val graduate = if (saved.profile?.type == StudentType.POSTGRADUATE) GraduateClient(saved) else null
private val undergraduate = if (graduate == null) CampusClient(saved) else null
val jar: SessionCookies get() = graduate?.jar ?: requireNotNull(undergraduate).jar

fun schedule(term: String? = null): Pair<Schedule, JSONObject> {
    graduate?.let {
        val raw = it.scheduleRaw(term)
        return GraduateClient.scheduleFromCache(raw) to JSONObject().put("graduate", raw)
    }
    return requireNotNull(undergraduate).schedule(term)
}
```

`CampusClient.kt` 的本科课表读取

```kotlin
fun schedule(semesterId: String? = null): Pair<Schedule, JSONObject> {
    var page: String
    try { page = get("${AuthPolicy.COURSE_ORIGIN}/student/for-std/course-table") }
    catch (_: CourseAuthRequired) { exchangeCourseTicket(); page = get("${AuthPolicy.COURSE_ORIGIN}/student/for-std/course-table") }
    val semesters = CampusParsers.semesters(page)
    require(semesters.isNotEmpty()) { "没有读取到学期信息" }
    val current = JSONObject(get("${AuthPolicy.COURSE_ORIGIN}/student/home/get-current-teach-week"))
    require(current.opt("currentSemester") is String || current.opt("currentSemester") is Number) { "校方学期数据格式已变化" }
    val currentName = current.optString("currentSemester").trim()
    require(currentName.isNotBlank() && currentName != "null") { "校方未返回当前学期" }
    val selected = if (semesterId != null) semesters.firstOrNull { it.id == semesterId }
        else semesters.firstOrNull { it.name.trim() == currentName || it.id == currentName }
    if (selected == null && semesterId == null) {
        val cache = JSONObject().put("semestersHtml", page).put("current", current)
            .put("currentUnavailable", true).put("fetchedOn", LocalDate.now().toString())
        return scheduleFromCache(cache) to cache
    }
    requireNotNull(selected) { "未能确认当前学期" }
    val raw = get("${AuthPolicy.COURSE_ORIGIN}/student/for-std/course-table/semester/${selected.id}/print-data?semesterId=${selected.id}&hasExperiment=false")
    val start = try { JSONObject(get("${AuthPolicy.COURSE_ORIGIN}/student/ws/semester/get/${selected.id}")).optString("startDate") }
        catch (_: IOException) { "" }
    val cache = JSONObject().put("semestersHtml", page).put("semesterId", selected.id).put("current", current)
        .put("activities", raw).put("startDate", start).put("fetchedOn", LocalDate.now().toString())
    return scheduleFromCache(cache) to cache
}
```

`GraduateClient.kt` 的研究生课表读取与校历计算

```kotlin
fun scheduleRaw(termId: String? = null): JSONObject = authenticated {
    val profile = loadProfile()
    val termRows = rows(json("wdkbapp", "/modules/xskcb/kfdxnxqcx.do"), "kfdxnxqcx")
    val terms = semesters(termRows)
    if (terms.isEmpty()) throw GraduateServiceFailure("研究生课表暂未开放学期查询")
    val today = LocalDate.now()
    var calendar: JSONObject? = null
    val markedCurrent = objects(termRows).firstOrNull { it.optString("SFDQXQ") == "1" }
        ?.let { text(it, "XNXQDM", "DM") }
    val selected = if (termId != null || !markedCurrent.isNullOrBlank()) {
        val found = terms.firstOrNull { it.id == (termId ?: markedCurrent) }
            ?: throw GraduateServiceFailure("没有找到所选研究生学期")
        calendar = calendar(found.id)
        found
    } else {
        // The first option can be a future semester; use the server's actual calendar.
        terms.firstOrNull { term ->
            val candidate = calendar(term.id)
            if (containsDate(candidate, today)) { calendar = candidate; true } else false
        }
    }
    if (selected == null) return@authenticated encodeSchedule(Schedule(terms, Semester("", "当前学期"), 0, null,
        emptyList(), today, currentUnavailable = true))
    val activeCalendar = calendar ?: throw GraduateServiceFailure("未读取到研究生校历")
    val units = rows(json("wdkbapp", "/modules/xskcb/xsskjccx.do", mapOf("XNXQDM" to selected.id, "XH" to profile.number)), "xsskjccx")
    val weekly = rows(json("wdkbapp", "/modules/xskcb/xspkjgcx.do", mapOf("XNXQDM" to selected.id, "*order" to "-ZCBH")), "xspkjgcx")
    val dated = rows(json("wdkbapp", "/modules/xskcb/rqapjgcx.do", mapOf("XNXQDM" to selected.id)), "rqapjgcx")
    val lessons = parseLessons(units, weekly, dated, activeCalendar)
    val start = firstTeachingMonday(activeCalendar)
    if (start == null && lessons.isNotEmpty()) throw GraduateServiceFailure("研究生校历缺少教学周信息")
    val serverWeek = weekFor(activeCalendar, today) ?: start?.let { Math.floorDiv(ChronoUnit.DAYS.between(it, today), 7).toInt() + 1 } ?: 0
    encodeSchedule(Schedule(terms, selected, serverWeek, start, lessons, today))
}
```
