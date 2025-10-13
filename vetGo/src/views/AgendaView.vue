<script setup>
import { ref, computed, onMounted } from 'vue'
import ApiService from '@/stores/api.js'


const agendamentos = ref([])
const tutores = ref([])
const veterinarios = ref([])
const pets = ref([])
const servicos = ref([])

const loading = ref(false)
const creatingAgendamento = ref(false)
const estatisticasLoading = ref(false)
const showNovoAgendamento = ref(false)
const showEstatisticas = ref(false)
const estatisticasVeterinarios = ref([])
const processingAction = ref(null)
const metadata = ref(null)
const submitError = ref(null)
const successMessage = ref(null)

const filters = ref({
  tutor_id: '',
  veterinario_id: '',
  status: '',
  data_inicio: '',
  data_fim: ''
})

const novoAgendamento = ref({
  data_hora: '',
  pet: '',
  veterinario: '',
  servico: ''
})

const errors = ref({})


const minDateTime = computed(() => {
  const now = new Date()
 
  now.setMinutes(now.getMinutes() - now.getTimezoneOffset())
  return now.toISOString().slice(0, 16)
})


const showSuccess = (message) => {
  successMessage.value = message
  setTimeout(() => {
    successMessage.value = null
  }, 3000)
}

const showError = (message) => {
  submitError.value = message
  setTimeout(() => {
    submitError.value = null
  }, 5000)
}

const formatDateTime = (dateTimeString) => {
  if (!dateTimeString) return 'N/A'
  return new Date(dateTimeString).toLocaleString('pt-BR')
}

const getStatusDisplay = (status) => {
  const map = {
    pendente: 'Pendente',
    confirmado: 'Confirmado',
    cancelado: 'Cancelado'
  }
  return map[status] || status
}

const clearFilters = () => {
  filters.value = {
    tutor_id: '',
    veterinario_id: '',
    status: '',
    data_inicio: '',
    data_fim: ''
  }
  fetchAgendamentos()
}

const resetNovoAgendamento = () => {
  novoAgendamento.value = {
    data_hora: '',
    pet: '',
    veterinario: '',
    servico: ''
  }
}

const fecharModal = () => {
  showNovoAgendamento.value = false
  resetNovoAgendamento()
  errors.value = {}
  submitError.value = null
}

const fetchAgendamentos = async () => {
  loading.value = true
  submitError.value = null
  
  try {
    const data = await ApiService.getAgendamentos(filters.value)
    
   
    if (data && typeof data === 'object') {
      if (data.metadata) {
        agendamentos.value = data.results || data.data || []
        metadata.value = data.metadata
      } else if (data.results) {
        agendamentos.value = data.results
        metadata.value = { total: data.count }
      } else {
        agendamentos.value = Array.isArray(data) ? data : []
        metadata.value = null
      }
    } else {
      agendamentos.value = []
      metadata.value = null
    }
    
  } catch (err) {
    console.error('Erro ao buscar agendamentos:', err)
    showError('Erro ao carregar agendamentos. Verifique se o servidor está rodando.')
    agendamentos.value = []
  } finally {
    loading.value = false
  }
}

const fetchDadosAuxiliares = async () => {
  try {
    const [t, v, p, s] = await Promise.all([
      ApiService.getTutores(),
      ApiService.getVeterinarios(),
      ApiService.getPets(),
      ApiService.getServicos()
    ])
    tutores.value = t
    veterinarios.value = v
    pets.value = p
    servicos.value = s
  } catch (err) {
    console.error('Erro ao buscar dados auxiliares:', err)
    showError('Erro ao carregar dados auxiliares.')
  }
}

const fetchProximosAgendamentos = async () => {
  loading.value = true
  try {
    const data = await ApiService.getProximosAgendamentos()
    agendamentos.value = data
    metadata.value = null
    showSuccess('Próximos agendamentos carregados!')
  } catch (err) {
    console.error('Erro ao buscar próximos agendamentos:', err)
    showError('Erro ao buscar próximos agendamentos.')
  } finally {
    loading.value = false
  }
}

const fetchAgendamentosPorVeterinario = async () => {
  estatisticasLoading.value = true
  showEstatisticas.value = true
  try {
    estatisticasVeterinarios.value = await ApiService.getAgendamentosPorVeterinario()
  } catch (err) {
    console.error('Erro ao buscar estatísticas:', err)
    showError('Erro ao carregar estatísticas.')
  } finally {
    estatisticasLoading.value = false
  }
}

const criarAgendamento = async () => {
  creatingAgendamento.value = true
  errors.value = {}
  submitError.value = null

  try {
    const payload = {
      data_hora: novoAgendamento.value.data_hora,
      pet: parseInt(novoAgendamento.value.pet),
      veterinario: parseInt(novoAgendamento.value.veterinario),
      servico: parseInt(novoAgendamento.value.servico),
      status: 'pendente'
    }
    
    await ApiService.createAgendamento(payload)
    fecharModal()
    fetchAgendamentos()
    showSuccess('Agendamento criado com sucesso!')
  } catch (err) {
    console.error('Erro ao criar agendamento:', err)
    showError('Erro ao criar agendamento. Verifique os dados e tente novamente.')
  } finally {
    creatingAgendamento.value = false
  }
}

const confirmarAgendamento = async (id) => {
  processingAction.value = id
  try {
    await ApiService.confirmAgendamento(id)
    fetchAgendamentos()
    showSuccess('Agendamento confirmado com sucesso!')
  } catch (err) {
    console.error('Erro ao confirmar agendamento:', err)
    showError('Erro ao confirmar agendamento.')
  } finally {
    processingAction.value = null
  }
}

const cancelarAgendamento = async (id) => {
  if (!confirm('Tem certeza que deseja cancelar este agendamento?')) return
  
  processingAction.value = id
  try {
    await ApiService.cancelAgendamento(id)
    fetchAgendamentos()
    showSuccess('Agendamento cancelado com sucesso!')
  } catch (err) {
    console.error('Erro ao cancelar agendamento:', err)
    showError('Erro ao cancelar agendamento.')
  } finally {
    processingAction.value = null
  }
}


onMounted(() => {
  fetchAgendamentos()
  fetchDadosAuxiliares()
})
</script>

<template>
  <div class="agendamentos-container">
    <h1>Agendamentos</h1>
    
   
    <div v-if="submitError" class="alert alert-error">
      {{ submitError }}
    </div>

    <div v-if="successMessage" class="alert alert-success">
      {{ successMessage }}
    </div>

  
    <div class="filters">
      <div class="filter-group">
        <label for="tutor">Tutor:</label>
        <select id="tutor" v-model="filters.tutor_id" @change="fetchAgendamentos">
          <option value="">Todos os tutores</option>
          <option v-for="tutor in tutores" :key="tutor.id" :value="tutor.id">
            {{ tutor.nome }}
          </option>
        </select>
      </div>

      <div class="filter-group">
        <label for="veterinario">Veterinário:</label>
        <select id="veterinario" v-model="filters.veterinario_id" @change="fetchAgendamentos">
          <option value="">Todos os veterinários</option>
          <option v-for="vet in veterinarios" :key="vet.id" :value="vet.id">
            {{ vet.nome_completo }} - {{ vet.especialidade }}
          </option>
        </select>
      </div>

      <div class="filter-group">
        <label for="status">Status:</label>
        <select id="status" v-model="filters.status" @change="fetchAgendamentos">
          <option value="">Todos os status</option>
          <option value="pendente">Pendente</option>
          <option value="confirmado">Confirmado</option>
          <option value="cancelado">Cancelado</option>
        </select>
      </div>

      <div class="filter-group">
        <label for="data_inicio">Data Início:</label>
        <input type="datetime-local" id="data_inicio" v-model="filters.data_inicio" @change="fetchAgendamentos">
      </div>

      <div class="filter-group">
        <label for="data_fim">Data Fim:</label>
        <input type="datetime-local" id="data_fim" v-model="filters.data_fim" @change="fetchAgendamentos">
      </div>

      <button @click="clearFilters" class="btn-clear">Limpar Filtros</button>
    </div>

   
    <div class="actions">
      <button @click="showNovoAgendamento = true" class="btn-primary">
        Novo Agendamento
      </button>
      <button @click="fetchProximosAgendamentos" class="btn-secondary">
        Próximos Agendamentos
      </button>
      <button @click="fetchAgendamentosPorVeterinario" class="btn-secondary">
        Estatísticas por Veterinário
      </button>
    </div>

   
    <div v-if="metadata" class="metadata">
      <p><strong>Total:</strong> {{ metadata.total }} agendamentos</p>
      <p><strong>Última atualização:</strong> {{ formatDateTime(metadata.timestamp) }}</p>
    </div>

    
    <div class="agendamentos-list">
      <div v-if="loading" class="loading">
        <div class="spinner"></div>
        Carregando agendamentos...
      </div>
      
      <div v-else-if="agendamentos.length === 0" class="no-data">
      
        <p>Nenhum agendamento encontrado.</p>
        <button @click="clearFilters" class="btn-primary">Limpar Filtros</button>
      </div>

      <div v-else class="agendamentos-grid">
        <div v-for="agendamento in agendamentos" :key="agendamento.id" class="agendamento-card">
          <div class="agendamento-header">
            <h3>{{ formatDateTime(agendamento.data_hora) }}</h3>
            <span :class="`status status-${agendamento.status}`">
              {{ getStatusDisplay(agendamento.status) }}
            </span>
          </div>
          
          <div class="agendamento-info">
            <div class="info-row">
              <strong>Pet:</strong> 
              <span>{{ agendamento.pet_info?.nome || 'N/A' }}</span>
            </div>
            <div class="info-row">
              <strong>Tutor:</strong> 
              <span>{{ agendamento.tutor_info?.nome || 'N/A' }}</span>
            </div>
            <div class="info-row">
              <strong>Veterinário:</strong> 
              <span>{{ agendamento.veterinario_info?.nome_completo || 'N/A' }}</span>
            </div>
            <div class="info-row">
              <strong>Serviço:</strong> 
              <span>{{ agendamento.servico_info?.nome || 'N/A' }}</span>
            </div>
          </div>

          <div class="agendamento-actions">
            <button 
              v-if="agendamento.status === 'pendente'" 
              @click="confirmarAgendamento(agendamento.id)"
              class="btn-success"
              :disabled="processingAction === agendamento.id"
            >
              <span v-if="processingAction === agendamento.id" class="spinner-small"></span>
              {{ processingAction === agendamento.id ? 'Confirmando...' : 'Confirmar' }}
            </button>
            <button 
              v-if="agendamento.status !== 'cancelado'" 
              @click="cancelarAgendamento(agendamento.id)"
              class="btn-danger"
              :disabled="processingAction === agendamento.id"
            >
              <span v-if="processingAction === agendamento.id" class="spinner-small"></span>
              {{ processingAction === agendamento.id ? 'Cancelando...' : 'Cancelar' }}
            </button>
          </div>
        </div>
      </div>
    </div>

    
    <div v-if="showNovoAgendamento" class="modal-overlay" @click.self="fecharModal">
      <div class="modal">
        <div class="modal-header">
          <h2>Novo Agendamento</h2>
          <button @click="fecharModal" class="btn-close">&times;</button>
        </div>
        
        <form @submit.prevent="criarAgendamento">
          <div class="form-group">
            <label for="nova_data_hora">Data e Hora:</label>
            <input 
              type="datetime-local" 
              id="nova_data_hora" 
              v-model="novoAgendamento.data_hora" 
              required
              :min="minDateTime"
            >
            <small v-if="errors.data_hora" class="error">{{ errors.data_hora }}</small>
          </div>

          <div class="form-group">
            <label for="novo_pet">Pet:</label>
            <select id="novo_pet" v-model="novoAgendamento.pet" required>
              <option value="">Selecione um pet</option>
              <option v-for="pet in pets" :key="pet.id" :value="pet.id">
                {{ pet.nome }} (Tutor: {{ pet.tutor_nome || 'N/A' }})
              </option>
            </select>
            <small v-if="errors.pet" class="error">{{ errors.pet }}</small>
          </div>

          <div class="form-group">
            <label for="novo_veterinario">Veterinário:</label>
            <select id="novo_veterinario" v-model="novoAgendamento.veterinario" required>
              <option value="">Selecione um veterinário</option>
              <option v-for="vet in veterinarios" :key="vet.id" :value="vet.id">
                {{ vet.nome_completo }} - {{ vet.especialidade }}
              </option>
            </select>
            <small v-if="errors.veterinario" class="error">{{ errors.veterinario }}</small>
          </div>

          <div class="form-group">
            <label for="novo_servico">Serviço:</label>
            <select id="novo_servico" v-model="novoAgendamento.servico" required>
              <option value="">Selecione um serviço</option>
              <option v-for="servico in servicos" :key="servico.id" :value="servico.id">
                {{ servico.nome }} - R$ {{ servico.preco }}
              </option>
            </select>
            <small v-if="errors.servico" class="error">{{ errors.servico }}</small>
          </div>

          <div class="form-actions">
            <button type="button" @click="fecharModal" class="btn-secondary">
              Cancelar
            </button>
            <button type="submit" :disabled="creatingAgendamento" class="btn-primary">
              <span v-if="creatingAgendamento" class="spinner-small"></span>
              {{ creatingAgendamento ? 'Criando...' : 'Criar Agendamento' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    
    <div v-if="showEstatisticas" class="modal-overlay" @click.self="showEstatisticas = false">
      <div class="modal large-modal">
        <div class="modal-header">
          <h2>Agendamentos por Veterinário</h2>
          <button @click="showEstatisticas = false" class="btn-close">&times;</button>
        </div>
        
        <div v-if="estatisticasLoading" class="loading">
          <div class="spinner"></div>
          Carregando estatísticas...
        </div>
        
        <div v-else class="estatisticas-grid">
          <div v-for="estatistica in estatisticasVeterinarios" :key="estatistica.veterinario" class="estatistica-card">
            <h3>{{ estatistica.veterinario }}</h3>
            <p><strong>Especialidade:</strong> {{ estatistica.especialidade }}</p>
            <p><strong>Total:</strong> {{ estatistica.total_agendamentos }}</p>
            <p><strong>Confirmados:</strong> {{ estatistica.confirmados }}</p>
            <p><strong>Pendentes:</strong> {{ estatistica.pendentes }}</p>
          </div>
        </div>

        <div class="form-actions">
          <button @click="showEstatisticas = false" class="btn-primary">
            Fechar
          </button>
        </div>
      </div>
    </div>
  </div>
</template>


<style scoped>

.alert {
  padding: 1rem;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.alert-error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.alert-success {
  background: #d1edff;
  color: #7cab75;
  border: 1px solid #b3d9ff;
}

.spinner {
  border: 2px solid #f3f3f3;
  border-top: 2px solid #7cab75;
  border-radius: 50%;
  width: 20px;
  height: 20px;
  animation: spin 1s linear infinite;
  display: inline-block;
  margin-right: 10px;
}

.spinner-small {
  border: 2px solid #f3f3f3;
  border-top: 2px solid #ffffff;
  border-radius: 50%;
  width: 12px;
  height: 12px;
  animation: spin 1s linear infinite;
  display: inline-block;
  margin-right: 5px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #eee;
}

.btn-close {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: #6c757d;
}

.btn-close:hover {
  color: #333;
}

.no-data {
  text-align: center;
  padding: 3rem;
  color: #6c757d;
  font-size: 1.1rem;
  background: #f8f9fa;
  border-radius: 8px;
  border: 2px dashed #dee2e6;
}

.no-data p:first-child {
  font-size: 3rem;
  margin-bottom: 1rem;
}
</style>