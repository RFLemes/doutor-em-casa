# consulta-em-casa
Repositório para aplicativo Consulta em casa, projeto de extensão da Estácio
# Projeto: Consulta em Casa (Kotlin) - Código completo
---

@file:OptIn(androidx.compose.material3.ExperimentalMaterial3Api::class)

package com.example.consultaemcasa

import android.content.Context
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.enableEdgeToEdge
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.ArrowBack
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextOverflow
import androidx.compose.ui.unit.dp
import androidx.compose.ui.window.Dialog
import org.json.JSONArray
import org.json.JSONObject
import java.text.SimpleDateFormat
import java.util.*


data class Paciente(
    val id: String = UUID.randomUUID().toString(),
    var nome: String,
    var nascimento: String,
    var telefone: String,
    var endereco: String
)

data class Consulta(
    val id: String = UUID.randomUUID().toString(),
    val pacienteId: String,
    var dataHora: String,
    var observacoes: String,
    var realizado: Boolean = false
)

data class ProntuarioEntrada(
    val id: String = UUID.randomUUID().toString(),
    val pacienteId: String,
    var dataHora: String,
    var texto: String
)



private const val PREFS = "consulta_em_casa_prefs"
private const val KEY_PACIENTES = "pacientes"
private const val KEY_CONSULTAS = "consultas"
private const val KEY_PRONTUARIO = "prontuario"

object Repo {
    fun load(context: Context): Triple<MutableList<Paciente>, MutableList<Consulta>, MutableList<ProntuarioEntrada>> {
        val sp = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
        val pacientes = JSONArray(sp.getString(KEY_PACIENTES, "[]"))
        val consultas = JSONArray(sp.getString(KEY_CONSULTAS, "[]"))
        val prontuario = JSONArray(sp.getString(KEY_PRONTUARIO, "[]"))
        return Triple(
            (0 until pacientes.length()).map { jsonToPaciente(pacientes.getJSONObject(it)) }.toMutableList(),
            (0 until consultas.length()).map { jsonToConsulta(consultas.getJSONObject(it)) }.toMutableList(),
            (0 until prontuario.length()).map { jsonToProntuario(prontuario.getJSONObject(it)) }.toMutableList()
        )
    }

    fun save(context: Context, pacs: List<Paciente>, cons: List<Consulta>, pront: List<ProntuarioEntrada>) {
        val sp = context.getSharedPreferences(PREFS, Context.MODE_PRIVATE)
        sp.edit()
            .putString(KEY_PACIENTES, JSONArray(pacs.map { pacienteToJson(it) }).toString())
            .putString(KEY_CONSULTAS, JSONArray(cons.map { consultaToJson(it) }).toString())
            .putString(KEY_PRONTUARIO, JSONArray(pront.map { prontuarioToJson(it) }).toString())
            .apply()
    }

    private fun pacienteToJson(p: Paciente) = JSONObject().apply {
        put("id", p.id)
        put("nome", p.nome)
        put("nascimento", p.nascimento)
        put("telefone", p.telefone)
        put("endereco", p.endereco)
    }

    private fun consultaToJson(c: Consulta) = JSONObject().apply {
        put("id", c.id)
        put("pacienteId", c.pacienteId)
        put("dataHora", c.dataHora)
        put("observacoes", c.observacoes)
        put("realizado", c.realizado)
    }

    private fun prontuarioToJson(e: ProntuarioEntrada) = JSONObject().apply {
        put("id", e.id)
        put("pacienteId", e.pacienteId)
        put("dataHora", e.dataHora)
        put("texto", e.texto)
    }

    private fun jsonToPaciente(o: JSONObject) = Paciente(
        id = o.getString("id"),
        nome = o.getString("nome"),
        nascimento = o.getString("nascimento"),
        telefone = o.getString("telefone"),
        endereco = o.getString("endereco")
    )

    private fun jsonToConsulta(o: JSONObject) = Consulta(
        id = o.getString("id"),
        pacienteId = o.getString("pacienteId"),
        dataHora = o.getString("dataHora"),
        observacoes = o.getString("observacoes"),
        realizado = o.getBoolean("realizado")
    )

    private fun jsonToProntuario(o: JSONObject) = ProntuarioEntrada(
        id = o.getString("id"),
        pacienteId = o.getString("pacienteId"),
        dataHora = o.getString("dataHora"),
        texto = o.getString("texto")
    )
}



sealed class Screen {
    object Dashboard : Screen()
    object Pacientes : Screen()
    data class Prontuario(val pacienteId: String) : Screen()
    object NovaConsulta : Screen()
    data class EditarPaciente(val pacienteId: String) : Screen()
    object NovoPaciente : Screen()
}



class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent { App() }
    }
}



@Composable
fun App() {
    val ctx = LocalContext.current
    var pacientes by remember { mutableStateOf(mutableListOf<Paciente>()) }
    var consultas by remember { mutableStateOf(mutableListOf<Consulta>()) }
    var prontuarios by remember { mutableStateOf(mutableListOf<ProntuarioEntrada>()) }

    LaunchedEffect(Unit) {
        val (p, c, pr) = Repo.load(ctx)
        pacientes = p; consultas = c; prontuarios = pr
    }

    LaunchedEffect(pacientes, consultas, prontuarios) {
        Repo.save(ctx, pacientes, consultas, prontuarios)
    }

    var current by remember { mutableStateOf<Screen>(Screen.Dashboard) }

    MaterialTheme(colorScheme = lightColorScheme()) {
        Scaffold(topBar = { TopAppBar(title = { Text("Consulta em Casa") }) }) { inner ->
            Box(Modifier.padding(inner)) {
                when (val s = current) {
                    is Screen.Dashboard -> Dashboard(
                        pacientes = pacientes,
                        consultas = consultas,
                        onNovoPaciente = { current = Screen.NovoPaciente },
                        onNovaConsulta = { current = Screen.NovaConsulta },
                        onAbrirPacientes = { current = Screen.Pacientes },
                        onAbrirAgenda = { /* mantém mesma tela */ }
                    )
                    is Screen.Pacientes -> ListaPacientes(
                        pacientes = pacientes,
                        onVoltar = { current = Screen.Dashboard },
                        onEditar = { p -> current = Screen.EditarPaciente(p.id) },
                        onAbrirProntuario = { p -> current = Screen.Prontuario(p.id) },
                        onExcluir = { p -> pacientes = pacientes.toMutableList().also { it.removeAll { x -> x.id == p.id } } }
                    )
                    is Screen.NovoPaciente -> FormPaciente(
                        title = "Novo Paciente",
                        pacienteInicial = null,
                        onSalvar = { p -> pacientes = pacientes.toMutableList().also { it.add(p) }; current = Screen.Dashboard },
                        onCancelar = { current = Screen.Dashboard }
                    )
                    is Screen.EditarPaciente -> {
                        val existente = pacientes.find { it.id == s.pacienteId }
                        if (existente != null) FormPaciente(
                            title = "Editar Paciente",
                            pacienteInicial = existente,
                            onSalvar = { p -> pacientes = pacientes.toMutableList().also { val i = it.indexOfFirst { x -> x.id == p.id }; if (i>=0) it[i]=p }; current = Screen.Dashboard },
                            onCancelar = { current = Screen.Dashboard }
                        ) else Text("Paciente não encontrado")
                    }
                    is Screen.Prontuario -> ProntuarioScreen(
                        paciente = pacientes.find { it.id == s.pacienteId },
                        registros = prontuarios.filter { it.pacienteId == s.pacienteId },
                        onVoltar = { current = Screen.Pacientes },
                        onAdicionar = { texto -> prontuarios = prontuarios.toMutableList().also { it.add(ProntuarioEntrada(pacienteId = s.pacienteId, dataHora = agora(), texto = texto)) } },
                        onExcluir = { rid -> prontuarios = prontuarios.toMutableList().also { it.removeAll { x -> x.id == rid } } }
                    )
                    is Screen.NovaConsulta -> FormConsulta(
                        pacientes = pacientes,
                        onSalvar = { c -> consultas = consultas.toMutableList().also { it.add(c) }; current = Screen.Dashboard },
                        onCancelar = { current = Screen.Dashboard }
                    )
                }
            }
        }
    }
}



@Composable
fun Dashboard(
    pacientes: List<Paciente>,
    consultas: List<Consulta>,
    onNovoPaciente: () -> Unit,
    onNovaConsulta: () -> Unit,
    onAbrirPacientes: () -> Unit,
    onAbrirAgenda: () -> Unit
) {
    val proximas = consultas.sortedBy { parseDataHora(it.dataHora) }.take(5)

    Column(Modifier.fillMaxSize().padding(16.dp), verticalArrangement = Arrangement.spacedBy(16.dp)) {
        Text("Bem-vindo(a)", style = MaterialTheme.typography.headlineMedium)
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            EstatCard("Pacientes", pacientes.size.toString())
            EstatCard("Consultas", consultas.count { !it.realizado }.toString())
        }
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            Button(onClick = onNovoPaciente) { Text("Novo Paciente") }
            Button(onClick = onNovaConsulta) { Text("Nova Consulta") }
        }
        Divider()
        Text("Próximas consultas", style = MaterialTheme.typography.titleMedium)
        if (proximas.isEmpty()) {
            Text("Nenhuma consulta agendada.")
        } else {
            LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                items(proximas) { c ->
                    val nome = pacientes.find { it.id == c.pacienteId }?.nome ?: "(Desconhecido)"
                    Card(onClick = {}) {
                        Column(Modifier.padding(12.dp)) {
                            Text(nome, fontWeight = FontWeight.Bold)
                            Text(c.dataHora, maxLines = 1, overflow = TextOverflow.Ellipsis)
                            if (c.observacoes.isNotBlank()) Text(c.observacoes, maxLines = 1, overflow = TextOverflow.Ellipsis)
                        }
                    }
                }
            }
        }
        Spacer(Modifier.weight(1f))
        Row(horizontalArrangement = Arrangement.spacedBy(12.dp)) {
            OutlinedButton(onClick = onAbrirPacientes) { Text("Pacientes") }
            OutlinedButton(onClick = onAbrirAgenda) { Text("Agenda") }
        }
    }
}

@Composable
fun EstatCard(titulo: String, valor: String) {
    Card(modifier = Modifier.width(160.dp)) {
        Column(Modifier.padding(16.dp), horizontalAlignment = Alignment.CenterHorizontally) {
            Text(titulo)
            Text(valor, style = MaterialTheme.typography.headlineLarge, fontWeight = FontWeight.Bold)
        }
    }
}

@Composable
fun ListaPacientes(
    pacientes: List<Paciente>,
    onVoltar: () -> Unit,
    onEditar: (Paciente) -> Unit,
    onAbrirProntuario: (Paciente) -> Unit,
    onExcluir: (Paciente) -> Unit
) {
    var query by remember { mutableStateOf("") }
    val filtrados = pacientes.filter { it.nome.contains(query, ignoreCase = true) }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Pacientes") },
                navigationIcon = { IconButton(onClick = onVoltar) { Icon(imageVector = Icons.Filled.ArrowBack, contentDescription = "Voltar") } }
            )
        }
    ) { inner ->
        Column(Modifier.padding(inner).padding(12.dp)) {
            OutlinedTextField(
                value = query,
                onValueChange = { query = it },
                label = { Text("Buscar por nome") },
                modifier = Modifier.fillMaxWidth()
            )
            Spacer(Modifier.height(8.dp))
            LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                items(filtrados) { p ->
                    Card {
                        Column(Modifier.padding(12.dp)) {
                            Text(p.nome, fontWeight = FontWeight.Bold)
                            Text("Nasc.: ${p.nascimento}")
                            Text("Tel.: ${p.telefone}")
                            Text("End.: ${p.endereco}")
                            Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                                TextButton(onClick = { onAbrirProntuario(p) }) { Text("Prontuário") }
                                TextButton(onClick = { onEditar(p) }) { Text("Editar") }
                                TextButton(onClick = { onExcluir(p) }) { Text("Excluir") }
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun FormPaciente(
    title: String,
    pacienteInicial: Paciente? = null,
    onSalvar: (Paciente) -> Unit,
    onCancelar: () -> Unit
) {
    var nome by remember { mutableStateOf(pacienteInicial?.nome ?: "") }
    var nasc by remember { mutableStateOf(pacienteInicial?.nascimento ?: dataSimples()) }
    var tel by remember { mutableStateOf(pacienteInicial?.telefone ?: "") }
    var end by remember { mutableStateOf(pacienteInicial?.endereco ?: "") }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(title) },
                navigationIcon = { IconButton(onClick = onCancelar) { Icon(imageVector = Icons.Filled.ArrowBack, contentDescription = "Voltar") } }
            )
        }
    ) { inner ->
        Column(Modifier.padding(inner).padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
            OutlinedTextField(value = nome, onValueChange = { nome = it }, label = { Text("Nome completo") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = nasc, onValueChange = { nasc = it }, label = { Text("Nascimento (dd/MM/aaaa)") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = tel, onValueChange = { tel = it }, label = { Text("Telefone") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = end, onValueChange = { end = it }, label = { Text("Endereço") }, modifier = Modifier.fillMaxWidth(), minLines = 2)
            Spacer(Modifier.height(8.dp))
            Button(
                enabled = nome.isNotBlank(),
                onClick = {
                    val novo = (pacienteInicial?.copy(
                        nome = nome.trim(),
                        nascimento = nasc.trim(),
                        telefone = tel.trim(),
                        endereco = end.trim()
                    ) ?: Paciente(
                        nome = nome.trim(), nascimento = nasc.trim(), telefone = tel.trim(), endereco = end.trim()
                    ))
                    onSalvar(novo)
                }
            ) { Text("Salvar") }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun FormConsulta(
    pacientes: List<Paciente>,
    onSalvar: (Consulta) -> Unit,
    onCancelar: () -> Unit
) {
    var pacienteIdx by remember { mutableStateOf(0) }
    var data by remember { mutableStateOf(dataSimples()) }
    var hora by remember { mutableStateOf("09:00") }
    var obs by remember { mutableStateOf("") }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Nova Consulta") },
                navigationIcon = { IconButton(onClick = onCancelar) { Icon(imageVector = Icons.Filled.ArrowBack, contentDescription = "Voltar") } }
            )
        }
    ) { inner ->
        Column(Modifier.padding(inner).padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
            if (pacientes.isEmpty()) Text("Cadastre um paciente antes.")
            ExposedDropdownMenuBoxSample(
                label = "Paciente",
                options = pacientes.map { it.nome },
                selectedIndex = pacienteIdx,
                onSelectedIndexChange = { pacienteIdx = it }
            )
            OutlinedTextField(value = data, onValueChange = { data = it }, label = { Text("Data (dd/MM/aaaa)") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = hora, onValueChange = { hora = it }, label = { Text("Hora (HH:mm)") }, modifier = Modifier.fillMaxWidth())
            OutlinedTextField(value = obs, onValueChange = { obs = it }, label = { Text("Observações") }, modifier = Modifier.fillMaxWidth(), minLines = 3)
            Button(
                enabled = pacientes.isNotEmpty(),
                onClick = {
                    val escolhido = pacientes.getOrNull(pacienteIdx) ?: return@Button
                    val c = Consulta(pacienteId = escolhido.id, dataHora = "$data $hora", observacoes = obs.trim())
                    onSalvar(c)
                }
            ) { Text("Agendar") }
        }
    }
}

@Composable
fun AgendaScreen(
    consultas: List<Consulta>,
    pacientes: List<Paciente>,
    onVoltar: () -> Unit,
    onMarcarRealizada: (String, Boolean) -> Unit,
    onExcluir: (String) -> Unit
) {
    val ordenadas = consultas.sortedBy { parseDataHora(it.dataHora) }
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Agenda") },
                navigationIcon = { IconButton(onClick = onVoltar) { Icon(imageVector = Icons.Filled.ArrowBack, contentDescription = "Voltar") } }
            )
        }
    ) { inner ->
        if (ordenadas.isEmpty()) {
            Box(Modifier.padding(inner).fillMaxSize(), contentAlignment = Alignment.Center) {
                Text("Sem consultas agendadas.")
            }
        } else {
            LazyColumn(Modifier.padding(inner).padding(12.dp), verticalArrangement = Arrangement.spacedBy(8.dp)) {
                items(ordenadas) { c ->
                    val nome = pacientes.find { it.id == c.pacienteId }?.nome ?: "(Desconhecido)"
                    Card {
                        Column(Modifier.padding(12.dp)) {
                            Row(verticalAlignment = Alignment.CenterVertically, horizontalArrangement = Arrangement.SpaceBetween, modifier = Modifier.fillMaxWidth()) {
                                Column(Modifier.weight(1f)) {
                                    Text(nome, fontWeight = FontWeight.Bold)
                                    Text(c.dataHora)
                                    if (c.observacoes.isNotBlank()) Text(c.observacoes)
                                }
                                Column(horizontalAlignment = Alignment.CenterHorizontally) {
                                    Row(verticalAlignment = Alignment.CenterVertically) {
                                        Checkbox(checked = c.realizado, onCheckedChange = { onMarcarRealizada(c.id, it) })
                                        Text(if (c.realizado) "Realizada" else "Pendente")
                                    }
                                    TextButton(onClick = { onExcluir(c.id) }) { Text("Excluir") }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun ProntuarioScreen(
    paciente: Paciente?,
    registros: List<ProntuarioEntrada>,
    onVoltar: () -> Unit,
    onAdicionar: (String) -> Unit,
    onExcluir: (String) -> Unit
) {
    var novoTexto by remember { mutableStateOf("") }
    var confirm by remember { mutableStateOf<Pair<String, Boolean>?>(null) }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Prontuário - ${paciente?.nome ?: "Paciente"}") },
                navigationIcon = { IconButton(onClick = onVoltar) { Icon(imageVector = Icons.Filled.ArrowBack, contentDescription = "Voltar") } }
            )
        }
    ) { inner ->
        Column(Modifier.padding(inner).padding(12.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
            OutlinedTextField(
                value = novoTexto,
                onValueChange = { novoTexto = it },
                label = { Text("Nova anotação") },
                modifier = Modifier.fillMaxWidth(),
                minLines = 3
            )
            Button(enabled = novoTexto.isNotBlank(), onClick = { onAdicionar(novoTexto.trim()); novoTexto = "" }) { Text("Adicionar") }
            Divider()
            if (registros.isEmpty()) {
                Text("Sem registros ainda.")
            } else {
                LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                    items(registros.sortedByDescending { parseDataHora(it.dataHora) }) { r ->
                        Card {
                            Column(Modifier.padding(12.dp)) {
                                Text(r.dataHora, fontWeight = FontWeight.Bold)
                                Text(r.texto)
                                Row(horizontalArrangement = Arrangement.spacedBy(8.dp)) {
                                    TextButton(onClick = { confirm = r.id to true }) { Text("Excluir") }
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    if (confirm != null) {
        ConfirmDialog(
            title = "Excluir registro?",
            message = "Esta ação não pode ser desfeita.",
            onDismiss = { confirm = null },
            onConfirm = { onExcluir(confirm!!.first); confirm = null }
        )
    }
}


@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ExposedDropdownMenuBoxSample(
    label: String,
    options: List<String>,
    selectedIndex: Int,
    onSelectedIndexChange: (Int) -> Unit
) {
    var expanded by remember { mutableStateOf(false) }
    val selected = options.getOrNull(selectedIndex) ?: ""

    ExposedDropdownMenuBox(expanded = expanded, onExpandedChange = { expanded = !expanded }) {
        OutlinedTextField(
            value = selected,
            onValueChange = {},
            readOnly = true,
            label = { Text(label) },
            trailingIcon = { ExposedDropdownMenuDefaults.TrailingIcon(expanded = expanded) },
            modifier = Modifier.menuAnchor().fillMaxWidth()
        )
        ExposedDropdownMenu(expanded = expanded, onDismissRequest = { expanded = false }) {
            options.forEachIndexed { index, text ->
                DropdownMenuItem(text = { Text(text) }, onClick = {
                    onSelectedIndexChange(index)
                    expanded = false
                })
            }
        }
    }
}

@Composable
fun ConfirmDialog(title: String, message: String, onDismiss: () -> Unit, onConfirm: () -> Unit) {
    Dialog(onDismissRequest = onDismiss) {
        Card {
            Column(Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
                Text(title, style = MaterialTheme.typography.titleLarge)
                Text(message)
                Row(horizontalArrangement = Arrangement.spacedBy(8.dp), modifier = Modifier.align(Alignment.End)) {
                    TextButton(onClick = onDismiss) { Text("Cancelar") }
                    Button(onClick = onConfirm) { Text("Confirmar") }
                }
            }
        }
    }
}

private fun parseDataHora(s: String): Long {
    return try {
        SimpleDateFormat("dd/MM/yyyy HH:mm", Locale.getDefault()).parse(s)?.time ?: Long.MAX_VALUE
    } catch (e: Exception) {
        Long.MAX_VALUE
    }
}

private fun agora(): String = SimpleDateFormat("dd/MM/yyyy HH:mm", Locale.getDefault()).format(Date())

private fun dataSimples(date: Date = Date()): String = SimpleDateFormat("dd/MM/yyyy", Locale.getDefault()).format(date)
