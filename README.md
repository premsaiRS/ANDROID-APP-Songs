# ANDROID-APP-Songs
package com.example.ca2cse226

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.dp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewModelScope
import androidx.lifecycle.viewmodel.compose.viewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import com.example.ca2cse226.ui.theme.CA2Cse226Theme

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            CA2Cse226Theme {
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                    Box(modifier = Modifier.padding(innerPadding)) {
                        PlaylistScreen()
                    }
                }
            }
        }
    }
}
data class Song(
    val title: String,
    val genre: String,
    val mood: String
)
class SongRepository {
    suspend fun fetchSongs(): List<Song> {
        kotlinx.coroutines.delay(1500L) // Simulate network delay
        return listOf(
            Song("Blinding Lights", "Pop", "Energetic"),
            Song("Shape of You", "Pop", "Relaxed"),
            Song("Bohemian Rhapsody", "Rock", "Energetic"),
            Song("Hotel California", "Rock", "Relaxed"),
            Song("Someone Like You", "Pop", "Energetic")
        )
    }
}
class PlaylistViewModel : ViewModel() {
    private val repository = SongRepository()
    private val _allSongs = MutableStateFlow<List<Song>>(emptyList())
    private val _isLoading = MutableStateFlow(true)
    val isLoading: StateFlow<Boolean> = _isLoading
    val availableGenres = listOf("All", "Pop", "Rock")
    val availableMoods = listOf("All", "Energetic", "Relaxed")
    val selectedGenre = MutableStateFlow("All")
    val selectedMood = MutableStateFlow("All")
    init {
        viewModelScope.launch {
            _isLoading.value = true
            _allSongs.value = repository.fetchSongs()
            _isLoading.value = false
        }
    }
    val filteredSongs: StateFlow<List<Song>> = combine(
        _allSongs,
        selectedGenre,
        selectedMood
    ) { songs, genre, mood ->
        songs.filter { song ->
            (genre == "All" || song.genre == genre) &&
                    (mood == "All" || song.mood == mood)
        }
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = emptyList()
    )
    fun onGenreSelected(genre: String) {
        selectedGenre.value = genre
    }
    fun onMoodSelected(mood: String) {
        selectedMood.value = mood
    }
}
@Composable
fun PlaylistScreen(viewModel: PlaylistViewModel = viewModel()) {
    // Lifecycle-aware observation
    val isLoading by viewModel.isLoading.collectAsStateWithLifecycle()
    val selectedGenre by viewModel.selectedGenre.collectAsStateWithLifecycle()
    val selectedMood by viewModel.selectedMood.collectAsStateWithLifecycle()
    val filteredSongs by viewModel.filteredSongs.collectAsStateWithLifecycle()
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(text = "Smart Playlist Builder", style = MaterialTheme.typography.headlineMedium)
        Spacer(modifier = Modifier.height(16.dp))
        if (isLoading) {
            Box(modifier = Modifier.fillMaxSize(), contentAlignment = androidx.compose.ui.Alignment.Center) {
                CircularProgressIndicator()
            }
        } else {
            Row(
                modifier = Modifier.fillMaxWidth(),
                horizontalArrangement = Arrangement.spacedBy(8.dp)
            ) {
                Box(modifier = Modifier.weight(1f)) {
                    FilterDropdown(
                        label = "Genre",
                        options = viewModel.availableGenres,
                        selectedOption = selectedGenre,
                        onOptionSelected = viewModel::onGenreSelected
                    )
                }
                Box(modifier = Modifier.weight(1f)) {
                    FilterDropdown(
                        label = "Mood",
                        options = viewModel.availableMoods,
                        selectedOption = selectedMood,
                        onOptionSelected = viewModel::onMoodSelected
                    )
                }
            }
            Spacer(modifier = Modifier.height(16.dp))
            Text(text = "Songs (${filteredSongs.size})", style = MaterialTheme.typography.titleMedium)
            Spacer(modifier = Modifier.height(8.dp))
            SongList(songs = filteredSongs)
        }
    }
}
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FilterDropdown(
    label: String,
    options: List<String>,
    selectedOption: String,
    onOptionSelected: (String) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }
    ExposedDropdownMenuBox(
        expanded = expanded,
        onExpandedChange = { expanded = !expanded }
    ) {
        OutlinedTextField(
            value = selectedOption,
            onValueChange = {},
            readOnly = true,
            label = { Text(label) },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = expanded) },
            modifier = Modifier.menuAnchor(ExposedDropdownMenuAnchorType.PrimaryNotEditable)
        )
        ExposedDropdownMenu(
            expanded = expanded,
            onDismissRequest = { expanded = false }
        ) {
            options.forEach { option ->
                DropdownMenuItem(
                    text = { Text(option) },
                    onClick = {
                        onOptionSelected(option)
                        expanded = false
                    }
                )
            }
        }
    }
}
@Composable
fun SongList(songs: List<Song>) {
    if (songs.isEmpty()) {
        Text(
            text = "No songs match the selected filters.",
            style = MaterialTheme.typography.bodyMedium,
            modifier = Modifier.padding(16.dp)
        )
    } else {
        LazyColumn(
            verticalArrangement = Arrangement.spacedBy(8.dp),
            modifier = Modifier.fillMaxSize()
        ) {
            items(songs) { song ->
                SongItem(song = song)
            }
        }
    }
}
@Composable
fun SongItem(song: Song) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = song.title, fontWeight = FontWeight.Bold, style = MaterialTheme.typography.bodyLarge)
            Spacer(modifier = Modifier.height(4.dp))
            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                AssistChip(
                    onClick = { },
                    label = { Text(song.genre) }
                )
                AssistChip(
                    onClick = { },
                    label = { Text(song.mood) }
                )
            }
        }
    }
}
