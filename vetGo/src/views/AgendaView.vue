<template>
  <div>
    <h1>Agendamentos</h1>
    
    <!-- Filtros -->
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
    </div>

    <!-- Botões de ação -->
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

    <!-- Lista de agendamentos -->
    <div class="agendamentos-list">
      <div v-if="loading" class="loading">Carregando...</div>
      
      <div v-else-if="agendamentos.length === 0" class="no-data">
        Nenhum agendamento encontrado.
      </div>

      <div v-else class="agendamentos-grid">
        <div v-for="agendamento in agendamentos" :key="agendamento.id" class="agendamento-card">
          <div class="agendamento-header">
            <h3>{{ formatDateTime(agendamento.data_hora) }}</h3>
            <span :class="`status status-${agendamento.status}`">
              {{ agendamento.status }}
            </span>
          </div>
          
          <div class="agendamento-info">
            <div class="info-row">
              <strong>Pet:</strong> {{ agendamento.pet_info.nome }}
            </div>
            <div class="info-row">
              <strong>Tutor:</strong> {{ agendamento.tutor_info.nome }}
            </div>
            <div class="info-row">
              <strong>Veterinário:</strong> {{ agendamento.veterinario_info.nome_completo }}
            </div>
            <div class="info-row">
              <strong>Serviço:</strong> {{ agendamento.servico_info.nome }}
            </div>
          </div>

          <div class="agendamento-actions">
            <button 
              v-if="agendamento.status === 'pendente'" 
              @click="confirmarAgendamento(agendamento.id)"
              class="btn-success"
            >
              Confirmar
            </button>
            <button 
              v-if="agendamento.status !== 'cancelado'" 
              @click="cancelarAgendamento(agendamento.id)"
              class="btn-danger"
            >
              Cancelar
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Modal para novo agendamento -->
    <div v-if="showNovoAgendamento" class="modal-overlay">
      <div class="modal">
        <h2>Novo Agendamento</h2>
        
        <form @submit.prevent="criarAgendamento">
          <div class="form-group">
            <label for="nova_data_hora">Data e Hora:</label>
            <input 
              type="datetime-local" 
              id="nova_data_hora" 
              v-model="novoAgendamento.data_hora" 
              required
            >
          </div>

          <div class="form-group">
            <label for="novo_pet">Pet:</label>
            <select id="novo_pet" v-model="novoAgendamento.pet" required>
              <option value="">Selecione um pet</option>
              <option v-for="pet in pets" :key="pet.id" :value="pet.id">
                {{ pet.nome }} ({{ pet.tutor_nome }})
              </option>
            </select>
          </div>

          <div class="form-group">
            <label for="novo_veterinario">Veterinário:</label>
            <select id="novo_veterinario" v-model="novoAgendamento.veterinario" required>
              <option value="">Selecione um veterinário</option>
              <option v-for="vet in veterinarios" :key="vet.id" :value="vet.id">
                {{ vet.nome_completo }} - {{ vet.especialidade }}
              </option>
            </select>
          </div>

          <div class="form-group">
            <label for="novo_servico">Serviço:</label>
            <select id="novo_servico" v-model="novoAgendamento.servico" required>
              <option value="">Selecione um serviço</option>
              <option v-for="servico in servicos" :key="servico.id" :value="servico.id">
                {{ servico.nome }} - R$ {{ servico.preco }}
              </option>
            </select>
          </div>

          <div class="form-actions">
            <button type="button" @click="showNovoAgendamento = false" class="btn-secondary">
              Cancelar
            </button>
            <button type="submit" class="btn-primary">
              Criar Agendamento
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- Modal de estatísticas -->
    <div v-if="showEstatisticas" class="modal-overlay">
      <div class="modal">
        <h2>Agendamentos por Veterinário</h2>
        
        <div class="estatisticas-grid">
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

<script>
export default {
  name: 'AgendamentosView',
  
  data() {
    return {
      agendamentos: [],
      tutores: [],
      veterinarios: [],
      pets: [],
      servicos: [],
      loading: false,
      showNovoAgendamento: false,
      showEstatisticas: false,
      estatisticasVeterinarios: [],
      
      filters: {
        tutor_id: '',
        veterinario_id: '',
        status: '',
        data_inicio: '',
        data_fim: ''
      },
      
      novoAgendamento: {
        data_hora: '',
        pet: '',
        veterinario: '',
        servico: ''
      }
    }
  },
  
  mounted() {
    this.fetchAgendamentos()
    this.fetchDadosAuxiliares()
  },
  
  methods: {
    async fetchAgendamentos() {
      this.loading = true
      try {
        const params = new URLSearchParams()
        
        Object.keys(this.filters).forEach(key => {
          if (this.filters[key]) {
            params.append(key, this.filters[key])
          }
        })
        
        const response = await fetch(`/api/agendamentos/?${params}`)
        const data = await response.json()
        
        this.agendamentos = Array.isArray(data) ? data : data.results || []
      } catch (error) {
        console.error('Erro ao buscar agendamentos:', error)
        this.agendamentos = []
      } finally {
        this.loading = false
      }
    },
    
    async fetchDadosAuxiliares() {
      try {
        // Buscar tutores, veterinários, pets e serviços
        const [tutoresRes, veterinariosRes, petsRes, servicosRes] = await Promise.all([
          fetch('/api/tutores/'),
          fetch('/api/veterinarios/'),
          fetch('/api/pets/'),
          fetch('/api/servicos/')
        ])
        
        this.tutores = await tutoresRes.json()
        this.veterinarios = await veterinariosRes.json()
        this.pets = await petsRes.json()
        this.servicos = await servicosRes.json()
      } catch (error) {
        console.error('Erro ao buscar dados auxiliares:', error)
      }
    },
    
    async fetchProximosAgendamentos() {
      this.loading = true
      try {
        const response = await fetch('/api/agendamentos/proximos/')
        this.agendamentos = await response.json()
      } catch (error) {
        console.error('Erro ao buscar próximos agendamentos:', error)
      } finally {
        this.loading = false
      }
    },
    
    async fetchAgendamentosPorVeterinario() {
      try {
        const response = await fetch('/api/agendamentos/por_veterinario/')
        this.estatisticasVeterinarios = await response.json()
        this.showEstatisticas = true
      } catch (error) {
        console.error('Erro ao buscar estatísticas:', error)
      }
    },
    
    async criarAgendamento() {
      try {
        const response = await fetch('/api/agendamentos/', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
          },
          body: JSON.stringify(this.novoAgendamento)
        })
        
        if (response.ok) {
          this.showNovoAgendamento = false
          this.resetNovoAgendamento()
          this.fetchAgendamentos()
          alert('Agendamento criado com sucesso!')
        } else {
          const error = await response.json()
          alert(`Erro ao criar agendamento: ${JSON.stringify(error)}`)
        }
      } catch (error) {
        console.error('Erro ao criar agendamento:', error)
        alert('Erro ao criar agendamento')
      }
    },
    
    async confirmarAgendamento(id) {
      try {
        const response = await fetch(`/api/agendamentos/${id}/confirmar/`, {
          method: 'POST'
        })
        
        if (response.ok) {
          this.fetchAgendamentos()
          alert('Agendamento confirmado!')
        }
      } catch (error) {
        console.error('Erro ao confirmar agendamento:', error)
      }
    },
    
    async cancelarAgendamento(id) {
      if (!confirm('Tem certeza que deseja cancelar este agendamento?')) {
        return
      }
      
      try {
        const response = await fetch(`/api/agendamentos/${id}/cancelar/`, {
          method: 'POST'
        })
        
        if (response.ok) {
          this.fetchAgendamentos()
          alert('Agendamento cancelado!')
        }
      } catch (error) {
        console.error('Erro ao cancelar agendamento:', error)
      }
    },
    
    resetNovoAgendamento() {
      this.novoAgendamento = {
        data_hora: '',
        pet: '',
        veterinario: '',
        servico: ''
      }
    },
    
    formatDateTime(dateTimeString) {
      return new Date(dateTimeString).toLocaleString('pt-BR')
    }
  }
}
</script>

<style scoped>
.filters {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
  flex-wrap: wrap;
  padding: 1rem;
  background: #f5f5f5;
  border-radius: 8px;
}

.filter-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.filter-group label {
  font-weight: bold;
  font-size: 0.9rem;
}

.actions {
  display: flex;
  gap: 1rem;
  margin-bottom: 2rem;
}

.btn-primary {
  background: #007bff;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.btn-secondary {
  background: #6c757d;
  color: white;
  border: none;
  padding: 0.5rem 1rem;
  border-radius: 4px;
  cursor: pointer;
}

.btn-success {
  background: #28a745;
  color: white;
  border: none;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  cursor: pointer;
}

.btn-danger {
  background: #dc3545;
  color: white;
  border: none;
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  cursor: pointer;
}

.agendamentos-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 1rem;
}

.agendamento-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
  background: white;
}

.agendamento-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1rem;
}

.status {
  padding: 0.25rem 0.5rem;
  border-radius: 4px;
  font-size: 0.8rem;
  font-weight: bold;
}

.status-pendente {
  background: #fff3cd;
  color: #856404;
}

.status-confirmado {
  background: #d1ecf1;
  color: #0c5460;
}

.status-cancelado {
  background: #f8d7da;
  color: #721c24;
}

.agendamento-info {
  margin-bottom: 1rem;
}

.info-row {
  margin-bottom: 0.5rem;
}

.agendamento-actions {
  display: flex;
  gap: 0.5rem;
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal {
  background: white;
  padding: 2rem;
  border-radius: 8px;
  max-width: 500px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}

.form-group input,
.form-group select {
  width: 100%;
  padding: 0.5rem;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.form-actions {
  display: flex;
  gap: 1rem;
  justify-content: flex-end;
  margin-top: 1rem;
}

.estatisticas-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 1rem;
  margin: 1rem 0;
}

.estatistica-card {
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 1rem;
  background: #f8f9fa;
}

.loading {
  text-align: center;
  padding: 2rem;
  font-size: 1.2rem;
}

.no-data {
  text-align: center;
  padding: 2rem;
  color: #6c757d;
}
</style>
