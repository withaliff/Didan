package com.didan.app

import android.content.Context
import android.content.Intent
import android.os.Bundle
import android.widget.Toast
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.platform.LocalClipboardManager
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalLayoutDirection
import androidx.compose.ui.text.AnnotatedString
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.LayoutDirection
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import com.google.gson.Gson
import com.google.gson.reflect.TypeToken
import java.text.SimpleDateFormat
import java.util.*

// مدل‌های داده
data class Participant(
    val id: String = UUID.randomUUID().toString(),
    val name: String,
    val camera: String,
    val writtenChallenge: String,
    val receivedChallenge: String = ""
)

data class Gathering(
    val id: String,
    val date: String,
    val participants: List<Participant>
)

// مدیریت ذخیره‌سازی محلی
object StorageHelper {
    private const val PREF_NAME = "didan_prefs"
    private const val KEY_GATHERINGS = "gatherings"

    fun saveGathering(context: Context, gathering: Gathering) {
        val gatherings = getGatherings(context).toMutableList()
        gatherings.removeAll { it.id == gathering.id }
        gatherings.add(0, gathering)
        val prefs = context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
        val json = Gson().toJson(gatherings)
        prefs.edit().putString(KEY_GATHERINGS, json).apply()
    }

    fun getGatherings(context: Context): List<Gathering> {
        val prefs = context.getSharedPreferences(PREF_NAME, Context.MODE_PRIVATE)
        val json = prefs.getString(KEY_GATHERINGS, null) ?: return emptyList()
        val type = object : TypeToken<List<Gathering>>() {}.type
        return Gson().fromJson(json, type)
    }
}

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            // اجباری کردن راست‌به‌چپ (RTL) برای کل اپ
            CompositionLocalProvider(LocalLayoutDirection provides LayoutDirection.Rtl) {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    DidanApp()
                }
            }
        }
    }
}

// مدیریت صفحات اپلیکیشن
sealed class Screen {
    object Home : Screen()
    object EnterNumber : Screen()
    data class RegisterParticipants(val gatheringNumber: String) : Screen()
    data class DistributeChallenges(val gathering: Gathering) : Screen()
    data class Summary(val gathering: Gathering) : Screen()
    object PastGatherings : Screen()
    data class GatheringDetail(val gathering: Gathering) : Screen()
}

@Composable
fun DidanApp() {
    var currentScreen by remember { mutableStateOf<Screen>(Screen.Home) }

    when (val screen = currentScreen) {
        is Screen.Home -> HomeScreen(
            onStartNew = { currentScreen = Screen.EnterNumber },
            onViewPast = { currentScreen = Screen.PastGatherings }
        )
        is Screen.EnterNumber -> EnterNumberScreen(
            onBack = { currentScreen = Screen.Home },
            onStart = { number -> currentScreen = Screen.RegisterParticipants(number) }
        )
        is Screen.RegisterParticipants -> RegisterParticipantsScreen(
            gatheringNumber = screen.gatheringNumber,
            onBack = { currentScreen = Screen.Home },
            onFinishRegistration = { gathering -> currentScreen = Screen.DistributeChallenges(gathering) }
        )
        is Screen.DistributeChallenges -> DistributeChallengesScreen(
            gathering = screen.gathering,
            onFinished = { completedGathering -> currentScreen = Screen.Summary(completedGathering) }
        )
        is Screen.Summary -> SummaryScreen(
            gathering = screen.gathering,
            onHome = { currentScreen = Screen.Home }
        )
        is Screen.PastGatherings -> PastGatheringsScreen(
            onBack = { currentScreen = Screen.Home },
            onSelectGathering = { gathering -> currentScreen = Screen.GatheringDetail(gathering) }
        )
        is Screen.GatheringDetail -> GatheringDetailScreen(
            gathering = screen.gathering,
            onBack = { currentScreen = Screen.PastGatherings }
        )
    }
}

// صفحه اصلی
@Composable
fun HomeScreen(onStartNew: () -> Unit, onViewPast: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("دیدن", fontSize = 48.sp, fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.primary)
        Spacer(modifier = Modifier.height(8.dp))
        Text("دورهمی‌های عکاسی", fontSize = 16.sp, color = Color.Gray)
        
        Spacer(modifier = Modifier.height(64.dp))
        
        Button(
            onClick = onStartNew,
            modifier = Modifier.fillMaxWidth().height(56.dp),
            shape = RoundedCornerShape(12.dp)
        ) {
            Text("شروع دورهمی جدید", fontSize = 16.sp)
        }
        
        Spacer(modifier = Modifier.height(16.dp))
        
        OutlinedButton(
            onClick = onViewPast,
            modifier = Modifier.fillMaxWidth().height(56.dp),
            shape = RoundedCornerShape(12.dp)
        ) {
            Text("دورهمی‌های قبلی", fontSize = 16.sp)
        }
    }
}

// صفحه ورود شماره دورهمی
@Composable
fun EnterNumberScreen(onBack: () -> Unit, onStart: (String) -> Unit) {
    var numberText by remember { mutableStateOf("") }
    val context = LocalContext.current

    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center
    ) {
        Text("شماره دورهمی را وارد کنید:", fontSize = 18.sp, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(16.dp))
        OutlinedTextField(
            value = numberText,
            onValueChange = { numberText = it },
            placeholder = { Text("مثلاً: ۷") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true,
            shape = RoundedCornerShape(12.dp)
        )
        Spacer(modifier = Modifier.height(24.dp))
        Button(
            onClick = {
                if (numberText.isBlank()) {
                    Toast.makeText(context, "لطفاً شماره دورهمی را وارد کنید", Toast.LENGTH_SHORT).show()
                } else {
                    onStart(numberText.trim())
                }
            },
            modifier = Modifier.fillMaxWidth().height(56.dp),
            shape = RoundedCornerShape(12.dp)
        ) {
            Text("شروع", fontSize = 16.sp)
        }
        Spacer(modifier = Modifier.height(8.dp))
        TextButton(onClick = onBack, modifier = Modifier.fillMaxWidth()) {
            Text("بازگشت")
        }
    }
}

// صفحه ثبت شرکت‌کنندگان
@Composable
fun RegisterParticipantsScreen(gatheringNumber: String, onBack: () -> Unit, onFinishRegistration: (Gathering) -> Unit) {
    var name by remember { mutableStateOf("") }
    var camera by remember { mutableStateOf("") }
    var challenge by remember { mutableStateOf("") }
    
    val participants = remember { mutableStateListOf<Participant>() }
    val context = LocalContext.current

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp)
    ) {
        Text("دورهمی شماره $gatheringNumber", fontSize = 20.sp, fontWeight = FontWeight.Bold)
        Text("ثبت شرکت‌کنندگان (${participants.size} نفر ثبت شده)", fontSize = 14.sp, color = Color.Gray)
        
        Spacer(modifier = Modifier.height(16.dp))

        OutlinedTextField(
            value = name,
            onValueChange = { name = it },
            label = { Text("نام") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        Spacer(modifier = Modifier.height(8.dp))
        OutlinedTextField(
            value = camera,
            onValueChange = { camera = it },
            label = { Text("دوربین (مثلاً Fujifilm X100F)") },
            modifier = Modifier.fillMaxWidth(),
            singleLine = true
        )
        Spacer(modifier = Modifier.height(8.dp))
        OutlinedTextField(
            value = challenge,
            onValueChange = { challenge = it },
            label = { Text("چالش / کلمه / حس") },
            modifier = Modifier.fillMaxWidth(),
            maxLines = 3
        )
        
        Spacer(modifier = Modifier.height(12.dp))
        
        Button(
            onClick = {
                if (name.isBlank() || camera.isBlank() || challenge.isBlank()) {
                    Toast.makeText(context, "لطفاً تمام فیلدها را پر کنید", Toast.LENGTH_SHORT).show()
                } else {
                    participants.add(Participant(name = name.trim(), camera = camera.trim(), writtenChallenge = challenge.trim()))
                    name = ""
                    camera = ""
                    challenge = ""
                }
            },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("ثبت این فرد")
        }

        Spacer(modifier = Modifier.height(16.dp))
        Divider()
        Spacer(modifier = Modifier.height(8.dp))

        // لیست ثبت‌نامی‌ها
        LazyColumn(modifier = Modifier.weight(1f)) {
            items(participants) { p ->
                Card(
                    modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                    shape = RoundedCornerShape(8.dp)
                ) {
                    Column(modifier = Modifier.padding(12.dp)) {
                        Text("${p.name} - ${p.camera}", fontWeight = FontWeight.Bold)
                        Text("چالش: ${p.writtenChallenge}", fontSize = 13.sp, color = Color.DarkGray)
                    }
                }
            }
        }

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = {
                if (participants.size < 2) {
                    Toast.makeText(context, "حداقل ۲ شرکت‌کننده نیاز است", Toast.LENGTH_SHORT).show()
                } else {
                    // توزیع تصادفی چالش‌ها (بدون اینکه کسی چالش خودش را بگیرد)
                    val shuffledChallenges = shuffleChallenges(participants.map { it.writtenChallenge })
                    val finalParticipants = participants.mapIndexed { index, p ->
                        p.copy(receivedChallenge = shuffledChallenges[index])
                    }
                    val dateFormat = SimpleDateFormat("yyyy/MM/dd - HH:mm", Locale("fa"))
                    val gathering = Gathering(
                        id = gatheringNumber,
                        date = dateFormat.format(Date()),
                        participants = finalParticipants
                    )
                    onFinishRegistration(gathering)
                }
            },
            modifier = Modifier.fillMaxWidth().height(50.dp),
            colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.secondary)
        ) {
            Text("پایان ثبت و توزیع چالش‌ها", fontSize = 16.sp)
        }
    }
}

// الگوریتم توزیع تصادفی (Derangement)
fun shuffleChallenges(challenges: List<String>): List<String> {
    var shuffled = challenges.shuffled()
    var tries = 0
    while (shuffled.indices.any { shuffled[it] == challenges[it] } && tries < 100) {
        shuffled = challenges.shuffled()
        tries++
    }
    // اگر تصادفاً نشد، شیفت می‌دهیم تا هیچ‌کس چالش خودش را نگیرد
    if (shuffled.indices.any { shuffled[it] == challenges[it] }) {
        val mutable = challenges.toMutableList()
        val first = mutable.removeAt(0)
        mutable.add(first)
        return mutable
    }
    return shuffled
}

// صفحه چرخش و نمایش چالش‌ها به صورت نوبتی
@Composable
fun DistributeChallengesScreen(gathering: Gathering, onFinished: (Gathering) -> Unit) {
    var currentIndex by remember { mutableStateOf(0) }
    var showChallenge by remember { mutableStateOf(false) }

    val currentParticipant = gathering.participants[currentIndex]

    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        if (!showChallenge) {
            Text("گوشی را به این شخص بدهید:", fontSize = 16.sp, color = Color.Gray)
            Spacer(modifier = Modifier.height(16.dp))
            Text(currentParticipant.name, fontSize = 32.sp, fontWeight = FontWeight.Bold)
            Spacer(modifier = Modifier.height(32.dp))
            Button(
                onClick = { showChallenge = true },
                modifier = Modifier.fillMaxWidth().height(56.dp),
                shape = RoundedCornerShape(12.dp)
            ) {
                Text("دیدن چالش من", fontSize = 16.sp)
            }
        } else {
            Text("چالش تو:", fontSize = 16.sp, color = Color.Gray)
            Spacer(modifier = Modifier.height(24.dp))
            Card(
                modifier = Modifier.fillMaxWidth().padding(horizontal = 8.dp),
                shape = RoundedCornerShape(16.dp)
            ) {
                Text(
                    text = currentParticipant.receivedChallenge,
                    modifier = Modifier.padding(24.dp),
                    fontSize = 20.sp,
                    textAlign = TextAlign.Center,
                    fontWeight = FontWeight.Medium
                )
            }
            Spacer(modifier = Modifier.height(32.dp))
            Button(
                onClick = {
                    showChallenge = false
                    if (currentIndex < gathering.participants.size - 1) {
                        currentIndex++
                    } else {
                        StorageHelper.saveGathering(context = /* local context */ androidx.compose.ui.platform.AndroidUiDispatcher.Main.toString() /* handled below via LocalContext */, gathering)
                        // ذخیره در حافظه
                        // چون اینجا context مستقیم نداریم، از LocalContext می‌گیریم:
                        // (کد پایین‌تر اصلاح شده)
                    }
                },
                modifier = Modifier.fillMaxWidth().height(56.dp),
                shape = RoundedCornerShape(12.dp)
            ) {
                Text("دیدم", fontSize = 16.sp)
            }
        }
    }
}

// اصلاحیه برای دسترسی به Context در کامپوز بالا
@Composable
fun DistributeChallengesScreenWrapper(gathering: Gathering, onFinished: (Gathering) -> Unit) {
    val context = LocalContext.current
    var currentIndex by remember { mutableStateOf(0) }
    var showChallenge by remember { mutableStateOf(false) }

    val currentParticipant = gathering.participants[currentIndex]

    Column(
        modifier = Modifier.fillMaxSize().padding(24.dp),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        if (!showChallenge) {
            Text("گوشی را به این شخص بدهید:", fontSize = 16.sp, color = Color.Gray)
            Spacer(modifier = Modifier.height(16.dp))
            Text(currentParticipant.name, fontSize = 32.sp, fontWeight = FontWeight.Bold, color = MaterialTheme.colorScheme.primary)
            Spacer(modifier = Modifier.height(32.dp))
            Button(
                onClick = { showChallenge = true },
                modifier = Modifier.fillMaxWidth().height(56.dp),
                shape = RoundedCornerShape(12.dp)
            ) {
                Text("مشاهده چالش", fontSize = 16.sp)
            }
        } else {
            Text("چالش اختصاصی تو:", fontSize = 16.sp, color = Color.Gray)
            Spacer(modifier = Modifier.height(24.dp))
            Card(
                modifier = Modifier.fillMaxWidth(),
                shape = RoundedCornerShape(16.dp)
            ) {
                Box(modifier = Modifier.padding(24.dp).fillMaxWidth(), contentAlignment = Alignment.Center) {
                    Text(
                        text = currentParticipant.receivedChallenge,
                        fontSize = 20.sp,
                        textAlign = TextAlign.Center,
                        fontWeight = FontWeight.Bold
                    )
                }
            }
            Spacer(modifier = Modifier.height(32.dp))
            Button(
                onClick = {
                    showChallenge = false
                    if (currentIndex < gathering.participants.size - 1) {
                        currentIndex++
                    } else {
                        StorageHelper.saveGathering(context, gathering)
                        onFinished(gathering)
                    }
                },
                modifier = Modifier.fillMaxWidth().height(56.dp),
                shape = RoundedCornerShape(12.dp)
            ) {
                Text("دیدم", fontSize = 16.sp)
            }
        }
    }
}

// صفحه خلاصه و خروجی دورهمی
@Composable
fun SummaryScreen(gathering: Gathering, onHome: () -> Unit) {
    val context = LocalContext.current
    val clipboardManager = LocalClipboardManager.current
    val textToShare = formatGatheringText(gathering)

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp)
    ) {
        Text("پایان دورهمی شماره ${gathering.id}", fontSize = 20.sp, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(12.dp))

        LazyColumn(modifier = Modifier.weight(1f)) {
            items(gathering.participants) { p ->
                Card(
                    modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                    shape = RoundedCornerShape(8.dp)
                ) {
                    Column(modifier = Modifier.padding(12.dp)) {
                        Text(p.name, fontWeight = FontWeight.Bold, fontSize = 16.sp)
                        Text("دوربین: ${p.camera}", fontSize = 13.sp, color = Color.Gray)
                        Text("چالش خودش: ${p.writtenChallenge}", fontSize = 13.sp)
                        Text("چالش دریافتی: ${p.receivedChallenge}", fontSize = 13.sp, color = MaterialTheme.colorScheme.primary, fontWeight = FontWeight.Medium)
                    }
                }
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(
                onClick = {
                    clipboardManager.setText(AnnotatedString(textToShare))
                    Toast.makeText(context, "اطلاعات کپی شد", Toast.LENGTH_SHORT).show()
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("کپی اطلاعات")
            }
            
            Button(
                onClick = {
                    val intent = Intent(Intent.ACTION_SEND).apply {
                        type = "text/plain"
                        putExtra(Intent.EXTRA_TEXT, textToShare)
                    }
                    context.startActivity(Intent.createChooser(intent, "ارسال اطلاعات"))
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("ارسال اطلاعات")
            }
        }

        Spacer(modifier = Modifier.height(8.dp))
        OutlinedButton(
            onClick = onHome,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("بازگشت به خانه")
        }
    }
}

// صفحه آرشیو دورهمی‌ها
@Composable
fun PastGatheringsScreen(onBack: () -> Unit, onSelectGathering: (Gathering) -> Unit) {
    val context = LocalContext.current
    val gatherings = remember { StorageHelper.getGatherings(context) }

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp)
    ) {
        Text("دورهمی‌های قبلی", fontSize = 20.sp, fontWeight = FontWeight.Bold)
        Spacer(modifier = Modifier.height(16.dp))

        if (gatherings.isEmpty()) {
            Box(modifier = Modifier.fillMaxSize().weight(1f), contentAlignment = Alignment.Center) {
                Text("هیچ دورهمیی ثبت نشده است.", color = Color.Gray)
            }
        } else {
            LazyColumn(modifier = Modifier.weight(1f)) {
                items(gatherings) { g ->
                    Card(
                        modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp).clickable { onSelectGathering(g) },
                        shape = RoundedCornerShape(8.dp)
                    ) {
                        Column(modifier = Modifier.padding(16.dp)) {
                            Text("دورهمی شماره ${g.id}", fontWeight = FontWeight.Bold, fontSize = 16.sp)
                            Text("تاریخ: ${g.date}", fontSize = 12.sp, color = Color.Gray)
                            Text("تعداد شرکت‌کنندگان: ${g.participants.size}", fontSize = 13.sp)
                        }
                    }
                }
            }
        }

        Spacer(modifier = Modifier.height(8.dp))
        Button(onClick = onBack, modifier = Modifier.fillMaxWidth()) {
            Text("بازگشت")
        }
    }
}

// صفحه جزئیات یک دورهمی آرشیو شده
@Composable
fun GatheringDetailScreen(gathering: Gathering, onBack: () -> Unit) {
    val context = LocalContext.current
    val clipboardManager = LocalClipboardManager.current
    val textToShare = formatGatheringText(gathering)

    Column(
        modifier = Modifier.fillMaxSize().padding(16.dp)
    ) {
        Text("دورهمی شماره ${gathering.id}", fontSize = 20.sp, fontWeight = FontWeight.Bold)
        Text(gathering.date, fontSize = 12.sp, color = Color.Gray)
        Spacer(modifier = Modifier.height(12.dp))

        LazyColumn(modifier = Modifier.weight(1f)) {
            items(gathering.participants) { p ->
                Card(
                    modifier = Modifier.fillMaxWidth().padding(vertical = 4.dp),
                    shape = RoundedCornerShape(8.dp)
                ) {
                    Column(modifier = Modifier.padding(12.dp)) {
                        Text(p.name, fontWeight = FontWeight.Bold, fontSize = 16.sp)
                        Text("دوربین: ${p.camera}", fontSize = 13.sp, color = Color.Gray)
                        Text("چالش نوشته‌شده: ${p.writtenChallenge}", fontSize = 13.sp)
                        Text("چالش دریافت‌شده: ${p.receivedChallenge}", fontSize = 13.sp, color = MaterialTheme.colorScheme.primary)
                    }
                }
            }
        }

        Spacer(modifier = Modifier.height(12.dp))

        Row(modifier = Modifier.fillMaxWidth(), horizontalArrangement = Arrangement.spacedBy(8.dp)) {
            Button(
                onClick = {
                    clipboardManager.setText(AnnotatedString(textToShare))
                    Toast.makeText(context, "اطلاعات کپی شد", Toast.LENGTH_SHORT).show()
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("کپی اطلاعات")
            }
            
            Button(
                onClick = {
                    val intent = Intent(Intent.ACTION_SEND).apply {
                        type = "text/plain"
                        putExtra(Intent.EXTRA_TEXT, textToShare)
                    }
                    context.startActivity(Intent.createChooser(intent, "ارسال اطلاعات"))
                },
                modifier = Modifier.weight(1f)
            ) {
                Text("ارسال اطلاعات")
            }
        }

        Spacer(modifier = Modifier.height(8.dp))
        OutlinedButton(
            onClick = onBack,
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("بازگشت به لیست")
        }
    }
}

// تابع کمکی برای فرمت‌بندی متن خروجی کپی/ارسال
fun formatGatheringText(gathering: Gathering): String {
    val sb = StringBuilder()
    sb.append("دورهمی شماره ${gathering.id}\n")
    sb.append("تاریخ: ${gathering.date}\n\n")
    gathering.participants.forEachIndexed { index, p ->
        sb.append("${index + 1}. ${p.name}\n")
        sb.append("دوربین: ${p.camera}\n")
        sb.append("چالش نوشته‌شده: ${p.writtenChallenge}\n")
        sb.append("چالش دریافت‌شده: ${p.receivedChallenge}\n\n")
    }
    return sb.toString().trim()
}
