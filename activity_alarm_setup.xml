package com.example.nammarailubuddy

import android.content.Intent
import android.os.Bundle
import android.text.Editable
import android.text.TextWatcher
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.*
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import androidx.fragment.app.FragmentActivity
import com.google.firebase.database.ValueEventListener

class MainActivity : AppCompatActivity() {

    private lateinit var etStation: EditText
    private lateinit var rvSuggestions: RecyclerView
    private lateinit var rvTrains: RecyclerView
    private lateinit var tvSelectedStation: TextView
    private lateinit var tvNoTrains: TextView
    private lateinit var btnSetAlarm: Button

    private var selectedStation: Station? = null
    private var currentTrainAdapter: TrainAdapter? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        etStation = findViewById(R.id.etStation)
        rvSuggestions = findViewById(R.id.rvSuggestions)
        rvTrains = findViewById(R.id.rvTrains)
        tvSelectedStation = findViewById(R.id.tvSelectedStation)
        tvNoTrains = findViewById(R.id.tvNoTrains)
        btnSetAlarm = findViewById(R.id.btnSetAlarm)

        rvSuggestions.layoutManager = LinearLayoutManager(this)
        rvTrains.layoutManager = LinearLayoutManager(this)

        etStation.addTextChangedListener(object : TextWatcher {
            override fun afterTextChanged(s: Editable?) {
                val query = s.toString().trim()
                if (query.length >= 2) {
                    val results = TrainDataRepository.searchStations(query)
                    showStationSuggestions(results)
                } else {
                    rvSuggestions.visibility = View.GONE
                }
            }
            override fun beforeTextChanged(s: CharSequence?, start: Int, count: Int, after: Int) {}
            override fun onTextChanged(s: CharSequence?, start: Int, before: Int, count: Int) {}
        })

        btnSetAlarm.setOnClickListener {
            startActivity(Intent(this, AlarmSetupActivity::class.java))
        }
    }

    private fun showStationSuggestions(stations: List<Station>) {
        if (stations.isEmpty()) {
            rvSuggestions.visibility = View.GONE
            return
        }
        rvSuggestions.visibility = View.VISIBLE
        rvSuggestions.adapter = StationSuggestionAdapter(stations) { station ->
            selectedStation = station
            etStation.setText(station.name)
            rvSuggestions.visibility = View.GONE
            loadTrainsForStation(station)
        }
    }

    private fun loadTrainsForStation(station: Station) {
        tvSelectedStation.text = "Trains at ${station.name}"
        tvSelectedStation.visibility = View.VISIBLE

        val trains = TrainDataRepository.getTrainsForStation(station.code)
        if (trains.isEmpty()) {
            tvNoTrains.visibility = View.VISIBLE
            rvTrains.visibility = View.GONE
        } else {
            tvNoTrains.visibility = View.GONE
            rvTrains.visibility = View.VISIBLE

            // Clean up old adapter's listeners before replacing
            currentTrainAdapter?.removeAllListeners()

            val adapter = TrainAdapter(trains, station.code) { train, delayMinutes, correctedPlatform ->
                submitPing(train, station, delayMinutes, correctedPlatform)
            }
            currentTrainAdapter = adapter
            rvTrains.adapter = adapter
        }
    }

    private fun submitPing(train: Train, station: Station, delayMinutes: Int, correctedPlatform: Int) {
        val ping = PlatformPing(
            trainNumber = train.trainNumber,
            stationCode = station.code,
            platform = train.platformNumber,
            correctedPlatform = correctedPlatform,
            reportedBy = "community",
            timestamp = System.currentTimeMillis(),
            delayMinutes = delayMinutes
        )
        FirebaseHelper.submitPlatformPing(ping,
            onSuccess = {
                val msg = when {
                    correctedPlatform > 0 -> "Platform correction submitted! 🚉"
                    delayMinutes > 0      -> "Delay reported — $delayMinutes min late ⚠"
                    else                  -> "Thanks! On-time report submitted ✅"
                }
                Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()
                // No manual refresh needed — Firebase listener fires automatically
            },
            onError = { err ->
                Toast.makeText(this, "Error: $err", Toast.LENGTH_SHORT).show()
            }
        )
    }

    override fun onStop() {
        super.onStop()
        currentTrainAdapter?.removeAllListeners()
    }

    override fun onStart() {
        super.onStart()
        // Re-attach listeners if a station was already selected
        selectedStation?.let { loadTrainsForStation(it) }
    }

    override fun onDestroy() {
        super.onDestroy()
        currentTrainAdapter?.removeAllListeners()
    }
}

// ---- Station Suggestion Adapter ----
class StationSuggestionAdapter(
    private val stations: List<Station>,
    private val onClick: (Station) -> Unit
) : RecyclerView.Adapter<StationSuggestionAdapter.VH>() {

    class VH(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val tvName: TextView = itemView.findViewById(R.id.tvStationName)
        val tvCode: TextView = itemView.findViewById(R.id.tvStationCode)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_station_suggestion, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        val s = stations[position]
        holder.tvName.text = s.name
        holder.tvCode.text = s.code
        holder.itemView.setOnClickListener { onClick(s) }
    }

    override fun getItemCount() = stations.size
}

// ---- Train Adapter ----
class TrainAdapter(
    private val trains: List<Train>,
    private val stationCode: String,
    private val onPingSubmit: (Train, Int, Int) -> Unit
) : RecyclerView.Adapter<TrainAdapter.VH>() {

    // KEY FIX: store every listener keyed by trainNumber so we can remove them cleanly
    private val activeListeners = mutableMapOf<String, ValueEventListener>()

    class VH(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val tvTrainName: TextView         = itemView.findViewById(R.id.tvTrainName)
        val tvTrainNumber: TextView       = itemView.findViewById(R.id.tvTrainNumber)
        val tvTiming: TextView            = itemView.findViewById(R.id.tvTiming)
        val tvPlatform: TextView          = itemView.findViewById(R.id.tvPlatform)
        val tvPlatformCorrected: TextView = itemView.findViewById(R.id.tvPlatformCorrected)
        val tvDelayStatus: TextView       = itemView.findViewById(R.id.tvDelayStatus)
        val tvPingCount: TextView         = itemView.findViewById(R.id.tvPingCount)
        val llCoachLayout: LinearLayout   = itemView.findViewById(R.id.llCoachLayout)
        val btnPingOnTime: Button         = itemView.findViewById(R.id.btnPingOnTime)
        val btnPingDelay: Button          = itemView.findViewById(R.id.btnPingDelay)
        val btnFixPlatform: Button        = itemView.findViewById(R.id.btnFixPlatform)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_train, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        val train = trains[position]

        // Static fields
        holder.tvTrainName.text   = train.trainName
        holder.tvTrainNumber.text = "#${train.trainNumber}"
        holder.tvTiming.text      = "${train.fromStation} ${train.departureTime}  →  ${train.toStation} ${train.arrivalTime}"
        holder.tvPlatform.text    = "🚉 Platform ${train.platformNumber}"

        // Tap the ping count row to view all community reports
        holder.tvPingCount.setOnClickListener {
            val activity = holder.itemView.context as? androidx.fragment.app.FragmentActivity ?: return@setOnClickListener
            PingDetailsBottomSheet.newInstance(stationCode, train.trainNumber, train.trainName)
                .show(activity.supportFragmentManager, "ping_details")
        }

        // KEY FIX: remove any existing listener for this train before adding a new one.
        // Without this, every RecyclerView rebind stacks a NEW listener on top of the old
        // one — causing duplicate callbacks and stale/conflicting UI updates.
        activeListeners[train.trainNumber]?.let {
            FirebaseHelper.removePingListener(stationCode, train.trainNumber, it)
        }

        // Attach fresh real-time listener and store it
        val listener = FirebaseHelper.listenForPings(stationCode, train.trainNumber) { pings ->
            val analysis = FirebaseHelper.analyzePings(pings, train.platformNumber)

            holder.tvPingCount.text = if (analysis.confirmedCount > 0)
                "👥 ${analysis.confirmedCount} community report(s)" else ""

            when {
                analysis.isDelayed && analysis.avgDelayMinutes > 0 -> {
                    holder.tvDelayStatus.visibility = View.VISIBLE
                    holder.tvDelayStatus.text = "⚠ Running ~${analysis.avgDelayMinutes} min late"
                    holder.tvDelayStatus.setTextColor(android.graphics.Color.parseColor("#B71C1C"))
                    holder.tvDelayStatus.setBackgroundColor(android.graphics.Color.parseColor("#FFEBEE"))
                }
                analysis.confirmedCount > 0 -> {
                    holder.tvDelayStatus.visibility = View.VISIBLE
                    holder.tvDelayStatus.text = "✅ Reported running on time"
                    holder.tvDelayStatus.setTextColor(android.graphics.Color.parseColor("#1B5E20"))
                    holder.tvDelayStatus.setBackgroundColor(android.graphics.Color.parseColor("#E8F5E9"))
                }
                else -> holder.tvDelayStatus.visibility = View.GONE
            }

            holder.tvPlatform.text = "🚉 Platform ${analysis.communityPlatform}"
            holder.tvPlatformCorrected.visibility =
                if (analysis.platformCorrected) View.VISIBLE else View.GONE
        }
        activeListeners[train.trainNumber] = listener

        // Coach layout
        holder.llCoachLayout.removeAllViews()
        train.coachLayout.forEach { coach ->
            val ctx = holder.itemView.context
            val color = when (coach) {
                "Engine" -> android.graphics.Color.parseColor("#BF360C")
                "Ladies" -> android.graphics.Color.parseColor("#AD1457")
                "GEN"    -> android.graphics.Color.parseColor("#1565C0")
                "SL"     -> android.graphics.Color.parseColor("#4A148C")
                else     -> android.graphics.Color.parseColor("#37474F")
            }
            val tv = TextView(ctx).apply {
                text = coach
                setPadding(20, 10, 20, 10)
                textSize = 11f
                setTextColor(android.graphics.Color.WHITE)
                background = ctx.getDrawable(R.drawable.shape_badge)?.mutate()?.also {
                    it.setTint(color)
                }
                layoutParams = LinearLayout.LayoutParams(
                    LinearLayout.LayoutParams.WRAP_CONTENT,
                    LinearLayout.LayoutParams.WRAP_CONTENT
                ).also { it.marginEnd = 6 }
            }
            holder.llCoachLayout.addView(tv)
        }

        // Buttons
        holder.btnPingOnTime.setOnClickListener {
            onPingSubmit(train, 0, -1)
        }

        holder.btnPingDelay.setOnClickListener {
            val options = arrayOf("5 min late", "10 min late", "15 min late", "30 min late")
            val delays  = intArrayOf(5, 10, 15, 30)
            android.app.AlertDialog.Builder(holder.itemView.context)
                .setTitle("How late is ${train.trainName}?")
                .setItems(options) { _, which -> onPingSubmit(train, delays[which], -1) }
                .show()
        }

        holder.btnFixPlatform.setOnClickListener {
            val platforms = arrayOf("Platform 1", "Platform 2", "Platform 3",
                "Platform 4", "Platform 5", "Platform 6")
            android.app.AlertDialog.Builder(holder.itemView.context)
                .setTitle("Which platform is the train actually on?")
                .setItems(platforms) { _, which -> onPingSubmit(train, 0, which + 1) }
                .show()
        }
    }

    // Called when a card scrolls off screen — remove its listener immediately
    override fun onViewRecycled(holder: VH) {
        super.onViewRecycled(holder)
        val pos = holder.adapterPosition
        if (pos in trains.indices) {
            val trainNumber = trains[pos].trainNumber
            activeListeners[trainNumber]?.let {
                FirebaseHelper.removePingListener(stationCode, trainNumber, it)
                activeListeners.remove(trainNumber)
            }
        }
    }

    // Remove every active listener (call when switching stations or going to background)
    fun removeAllListeners() {
        activeListeners.forEach { (trainNumber, listener) ->
            FirebaseHelper.removePingListener(stationCode, trainNumber, listener)
        }
        activeListeners.clear()
    }

    override fun getItemCount() = trains.size
}