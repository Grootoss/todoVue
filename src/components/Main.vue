<script lang="ts" setup>
import Button from "./Button.vue";
import Search from "./Search.vue";
import Select from "./Select.vue";
import Modal from "./UI/Modal.vue";
import Todo from "./Todo.vue";
import { ref, computed, watch, onMounted } from "vue";

interface TodoItem {
  id: number;
  title: string;
  completed: boolean;
}

interface TodoFromAPI {
  id: number;
  text: string;
  completed: number;
  created_at: string;
  updated_at: string;
}

type FilterType = "all" | "complete" | "incomplete";
type ModalMode = "add" | "edit";

const API_URL = import.meta.env.VITE_API_URL || 'http://localhost:4000/api';

const todos = ref<TodoItem[]>([]);
const loading = ref(true);
const error = ref<string | null>(null);
const searchQuery = ref("");
const filterType = ref<FilterType>("all");
const modalOpen = ref(false);
const modalMode = ref<ModalMode>("add");
const editingTodo = ref<TodoItem | null>(null);
const isDarkTheme = ref(false);

const loadTodos = async () => {
  try {
    loading.value = true;
    const response = await fetch(`${API_URL}/todos`);
    
    if (!response.ok) throw new Error('Ошибка загрузки');
    
    const data: TodoFromAPI[] = await response.json();
    todos.value = data.map(todo => ({
      id: todo.id,
      title: todo.text,
      completed: todo.completed === 1
    }));
    
    error.value = null;
  } catch (err) {
    error.value = 'Не удалось загрузить задачи';
    console.error(err);
  } finally {
    loading.value = false;
  }
};

const addTodoToServer = async (title: string) => {
  try {
    const response = await fetch(`${API_URL}/todos`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ text: title })
    });
    
    if (!response.ok) throw new Error('Ошибка создания');
    
    const newTodo: TodoFromAPI = await response.json();
    const formattedTodo: TodoItem = {
      id: newTodo.id,
      title: newTodo.text,
      completed: newTodo.completed === 1
    };
    
    todos.value = [formattedTodo, ...todos.value];
  } catch (err) {
    error.value = 'Не удалось создать задачу';
    console.error(err);
  }
};

const updateTodoOnServer = async (id: number, title: string, completed: boolean) => {
  try {
    const response = await fetch(`${API_URL}/todos/${id}`, {
      method: 'PUT',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ 
        text: title, 
        completed: completed ? 1 : 0 
      })
    });
    
    if (!response.ok) throw new Error('Ошибка обновления');
    
    const updatedTodo: TodoFromAPI = await response.json();
    return {
      id: updatedTodo.id,
      title: updatedTodo.text,
      completed: updatedTodo.completed === 1
    };
  } catch (err) {
    error.value = 'Не удалось обновить задачу';
    console.error(err);
    throw err;
  }
};

const deleteTodoFromServer = async (id: number) => {
  try {
    const response = await fetch(`${API_URL}/todos/${id}`, {
      method: 'DELETE'
    });
    
    if (!response.ok) throw new Error('Ошибка удаления');
  } catch (err) {
    error.value = 'Не удалось удалить задачу';
    console.error(err);
    throw err;
  }
};

const toggleTodoOnServer = async (id: number) => {
  try {
    const response = await fetch(`${API_URL}/todos/${id}/toggle`, {
      method: 'PATCH'
    });
    
    if (!response.ok) throw new Error('Ошибка переключения статуса');
    
    const updatedTodo: TodoFromAPI = await response.json();
    return {
      id: updatedTodo.id,
      title: updatedTodo.text,
      completed: updatedTodo.completed === 1
    };
  } catch (err) {
    error.value = 'Не удалось обновить статус';
    console.error(err);
    throw err;
  }
};

watch(isDarkTheme, (newValue) => {
  if (newValue) {
    document.body.classList.add("dark-theme");
  } else {
    document.body.classList.remove("dark-theme");
  }
});

const toggleTheme = () => {
  isDarkTheme.value = !isDarkTheme.value;
};

const filteredTodos = computed(() => {
  let filtered = todos.value;

  if (searchQuery.value) {
    filtered = filtered.filter((todo) =>
      todo.title.toLowerCase().includes(searchQuery.value.toLowerCase()),
    );
  }

  switch (filterType.value) {
    case "complete":
      return filtered.filter((todo) => todo.completed);
    case "incomplete":
      return filtered.filter((todo) => !todo.completed);
    default:
      return filtered;
  }
});

const handleModalOpen = (mode: ModalMode = "add", todo?: TodoItem) => {
  modalMode.value = mode;
  if (mode === "edit" && todo) {
    editingTodo.value = { ...todo };
  } else {
    editingTodo.value = null;
  }
  modalOpen.value = true;
};

const handleModalClose = () => {
  modalOpen.value = false;
  editingTodo.value = null;
};

const handleAddTodo = async (title: string) => {
  if (!title.trim()) return;
  await addTodoToServer(title);
};

const handleEditTodo = async (title: string) => {
  if (editingTodo.value && title.trim()) {
    await updateTodoOnServer(editingTodo.value.id, title, editingTodo.value.completed);
    await loadTodos();
  }
};

const handleDeleteTodo = async (id: number) => {
  await deleteTodoFromServer(id);
  todos.value = todos.value.filter((todo) => todo.id !== id);
};

const handleToggleTodo = async (id: number) => {
  try {
    const updatedTodo = await toggleTodoOnServer(id);
    todos.value = todos.value.map((todo) =>
      todo.id === id ? updatedTodo : todo
    );
  } catch (err) {
  }
};

const handleSearch = (query: string) => {
  searchQuery.value = query;
};

const handleFilterChange = (filter: FilterType) => {
  filterType.value = filter;
};

onMounted(() => {
  loadTodos();
});
</script>

<template>
  <main>
    <div v-if="loading" class="app__loading">
      <p>Загрузка задач...</p>
    </div>

    <div v-else-if="error" class="app__error">
      <p>{{ error }}</p>
      <button @click="loadTodos">Повторить</button>
    </div>

    <template v-else>
      <nav class="app__nav nav">
        <Search @search="handleSearch" />
        <Select
          :modelValue="filterType"
          @update:modelValue="handleFilterChange"
          @change="handleFilterChange"
        />
        <Button class="app__themeChange" @click="toggleTheme" />
      </nav>
      
      <div v-if="filteredTodos.length > 0">
        <Todo
          :todos="filteredTodos"
          @toggle="handleToggleTodo"
          @edit="handleModalOpen('edit', $event)"
          @delete="handleDeleteTodo"
        />
      </div>
      <div v-else class="app__empty">
        <img src="/images/empty.png" alt="No todos" class="app__empty-image" />
        <p class="app__empty-text">Empty...</p>
      </div>

      <button class="app__add" @click="handleModalOpen('add')"></button>
    </template>
  </main>

  <Modal
    :isOpen="modalOpen"
    :mode="modalMode"
    :todo="editingTodo"
    @close="handleModalClose"
    @add="handleAddTodo"
    @edit="handleEditTodo"
  />
</template>

<style scoped lang="scss">
@use "@/sass/_variables.scss" as *;

.app__nav {
  display: flex;
  gap: 16px;
  margin-bottom: 20px;
}

.app__themeChange {
  display: block;
  width: 38px;
  height: 38px;
  border: none;
  border-radius: 5px;
  background-color: $default-purple;
  background-image: url("/images/moon.svg");
  background-position: 50% 50%;
  background-repeat: no-repeat;
  cursor: pointer;

  &:hover {
    background-color: $primary-purple;
  }
}

.dark-theme .app__themeChange {
  background-image: url("/images/light.svg");
}

.app__empty {
  width: 221px;
  margin: 70px auto 0;
}

.app__empty-text {
  text-align: center;
}

.dark-theme .app__empty-text {
  color: $default-white;
}

.app__add {
  display: block;
  width: 50px;
  height: 50px;
  background-color: $default-purple;
  border: none;
  border-radius: 50%;
  background-image: url("/images/plus.svg");
  background-position: 50% 50%;
  background-repeat: no-repeat;
  cursor: pointer;

  &:hover {
    background-color: $primary-purple;
  }
}

.app__loading {
  text-align: center;
  padding: 40px;
  font-size: 18px;
  color: $default-purple;
}

.app__error {
  text-align: center;
  padding: 40px;
  font-size: 18px;
  color: #ff0000;;
}

.app__error button {
  margin-top: 10px;
  padding: 8px 16px;
  background-color: $default-purple;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
}
</style>
