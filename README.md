/* ═══════════════════════════════════════════════════════════════════════════ */
/* TASK MASTER - TO-DO LIST APPLICATION                                        */
/* JavaScript Application Logic                                                */
/* ═══════════════════════════════════════════════════════════════════════════ */

// ─────────────────────────────────────────────────────────────────────────────
// TASK MANAGER CLASS
// ─────────────────────────────────────────────────────────────────────────────

class TaskManager {
    constructor() {
        // Configuration
        this.storageKey = 'taskMasterData';
        this.currentFilter = 'all';
        this.tasks = [];

        // DOM Elements
        this.taskInput = document.getElementById('taskInput');
        this.prioritySelect = document.getElementById('prioritySelect');
        this.addBtn = document.getElementById('addBtn');
        this.tasksList = document.getElementById('tasksList');
        this.tasksSubtitle = document.getElementById('tasksSubtitle');
        this.clearCompletedBtn = document.getElementById('clearCompletedBtn');
        this.clearAllBtn = document.getElementById('clearAllBtn');
        this.notificationContainer = document.getElementById('notificationContainer');

        // Statistics Elements
        this.totalTasksEl = document.getElementById('totalTasks');
        this.completedTasksEl = document.getElementById('completedTasks');
        this.pendingTasksEl = document.getElementById('pendingTasks');

        // Initialize
        this.init();
    }

    // ─────────────────────────────────────────────────────────────────────────
    // INITIALIZATION
    // ─────────────────────────────────────────────────────────────────────────

    init() {
        // Load tasks from local storage
        this.loadTasks();

        // Setup event listeners
        this.setupEventListeners();

        // Initial render
        this.render();

        // Log startup
        console.log('✓ Task Master initialized');
    }

    setupEventListeners() {
        // Add task button
        this.addBtn.addEventListener('click', () => this.addTask());

        // Input field - Enter key
        this.taskInput.addEventListener('keypress', (e) => {
            if (e.key === 'Enter') {
                e.preventDefault();
                this.addTask();
            }
        });

        // Clear buttons
        this.clearCompletedBtn.addEventListener('click', () => this.clearCompleted());
        this.clearAllBtn.addEventListener('click', () => this.clearAll());

        // Filter buttons
        document.querySelectorAll('.filter-btn').forEach(btn => {
            btn.addEventListener('click', (e) => {
                const filter = e.currentTarget.dataset.filter;
                this.setFilter(filter);
            });
        });
    }

    // ─────────────────────────────────────────────────────────────────────────
    // TASK OPERATIONS (CRUD)
    // ─────────────────────────────────────────────────────────────────────────

    addTask() {
        const text = this.taskInput.value.trim();
        const priority = this.prioritySelect.value;

        // Validation
        if (!text) {
            this.showNotification('Please enter a task', 'error');
            return;
        }

        if (text.length > 200) {
            this.showNotification('Task is too long (max 200 characters)', 'error');
            return;
        }

        // Create task object
        const task = {
            id: Date.now(),
            text: this.escapeHTML(text),
            completed: false,
            priority: priority,
            createdAt: new Date().toLocaleDateString('en-US', {
                month: '2-digit',
                day: '2-digit',
                year: 'numeric'
            }),
            createdTime: new Date().toLocaleTimeString('en-US', {
                hour: '2-digit',
                minute: '2-digit',
                hour12: true
            })
        };

        // Add to tasks array
        this.tasks.unshift(task);

        // Clear input
        this.taskInput.value = '';
        this.prioritySelect.value = 'medium';

        // Save and render
        this.saveTasks();
        this.render();

        // Show notification
        this.showNotification(`✓ Task added: "${text.substring(0, 30)}..."`, 'success');
    }

    deleteTask(id) {
        const taskIndex = this.tasks.findIndex(t => t.id === id);
        if (taskIndex === -1) return;

        const taskText = this.tasks[taskIndex].text;
        this.tasks.splice(taskIndex, 1);

        this.saveTasks();
        this.render();
        this.showNotification(`✓ Task deleted`, 'info');
    }

    toggleTask(id) {
        const task = this.tasks.find(t => t.id === id);
        if (task) {
            task.completed = !task.completed;
            this.saveTasks();
            this.render();
        }
    }

    editTask(id) {
        const task = this.tasks.find(t => t.id === id);
        if (!task) return;

        // Simple prompt for edit
        const newText = prompt('Edit task:', this.unescapeHTML(task.text));

        if (newText === null) return; // User cancelled
        if (!newText.trim()) {
            this.showNotification('Task cannot be empty', 'error');
            return;
        }
        if (newText.length > 200) {
            this.showNotification('Task is too long', 'error');
            return;
        }

        task.text = this.escapeHTML(newText.trim());
        this.saveTasks();
        this.render();
        this.showNotification('✓ Task updated', 'success');
    }

    // ─────────────────────────────────────────────────────────────────────────
    // FILTERING & SORTING
    // ─────────────────────────────────────────────────────────────────────────

    setFilter(filter) {
        this.currentFilter = filter;

        // Update filter button states
        document.querySelectorAll('.filter-btn').forEach(btn => {
            btn.classList.remove('active');
            if (btn.dataset.filter === filter) {
                btn.classList.add('active');
            }
        });

        this.render();
    }

    getFilteredTasks() {
        switch (this.currentFilter) {
            case 'active':
                return this.tasks.filter(t => !t.completed);
            case 'completed':
                return this.tasks.filter(t => t.completed);
            case 'high':
                return this.tasks.filter(t => t.priority === 'high');
            default:
                return this.tasks;
        }
    }

    // ─────────────────────────────────────────────────────────────────────────
    // RENDERING
    // ─────────────────────────────────────────────────────────────────────────

    render() {
        this.renderTasks();
        this.updateStats();
        this.updateFilterCounts();
    }

    renderTasks() {
        const filteredTasks = this.getFilteredTasks();
        this.tasksList.innerHTML = '';

        if (filteredTasks.length === 0) {
            this.tasksSubtitle.textContent = this.getEmptyMessage();
            return;
        }

        this.tasksSubtitle.textContent = `Showing ${filteredTasks.length} task${filteredTasks.length !== 1 ? 's' : ''}`;

        filteredTasks.forEach(task => {
            const taskElement = this.createTaskElement(task);
            this.tasksList.appendChild(taskElement);
        });
    }

    createTaskElement(task) {
        const div = document.createElement('div');
        div.className = `task-item ${task.priority}-priority`;
        if (task.completed) div.classList.add('completed');

        const priorityIcon = {
            high: '🔴',
            medium: '🟡',
            low: '🟢'
        }[task.priority];

        const priorityLabel = {
            high: 'High',
            medium: 'Medium',
            low: 'Low'
        }[task.priority];

        div.innerHTML = `
            <input 
                type="checkbox" 
                class="task-checkbox" 
                ${task.completed ? 'checked' : ''}
                aria-label="Mark task as complete"
            >
            <div class="task-content">
                <p class="task-text">${task.text}</p>
                <div class="task-meta">
                    <span class="task-priority ${task.priority}">
                        ${priorityIcon} ${priorityLabel}
                    </span>
                    <span class="task-date">${task.createdAt}</span>
                    <span class="task-time">${task.createdTime}</span>
                </div>
            </div>
            <div class="task-actions">
                <button class="task-btn edit-btn" title="Edit task" aria-label="Edit task">
                    ✏️
                </button>
                <button class="task-btn delete-btn" title="Delete task" aria-label="Delete task">
                    🗑️
                </button>
            </div>
        `;

        // Event listeners
        const checkbox = div.querySelector('.task-checkbox');
        const editBtn = div.querySelector('.edit-btn');
        const deleteBtn = div.querySelector('.delete-btn');

        checkbox.addEventListener('change', () => this.toggleTask(task.id));
        editBtn.addEventListener('click', () => this.editTask(task.id));
        deleteBtn.addEventListener('click', () => this.deleteTask(task.id));

        return div;
    }

    updateStats() {
        const completed = this.tasks.filter(t => t.completed).length;
        const pending = this.tasks.length - completed;

        this.totalTasksEl.textContent = this.tasks.length;
        this.completedTasksEl.textContent = completed;
        this.pendingTasksEl.textContent = pending;
    }

    updateFilterCounts() {
        const allCount = this.tasks.length;
        const activeCount = this.tasks.filter(t => !t.completed).length;
        const completedCount = this.tasks.filter(t => t.completed).length;
        const highCount = this.tasks.filter(t => t.priority === 'high').length;

        document.getElementById('count-all').textContent = allCount;
        document.getElementById('count-active').textContent = activeCount;
        document.getElementById('count-completed').textContent = completedCount;
        document.getElementById('count-high').textContent = highCount;
    }

    // ─────────────────────────────────────────────────────────────────────────
    // BULK OPERATIONS
    // ─────────────────────────────────────────────────────────────────────────

    clearCompleted() {
        const completedCount = this.tasks.filter(t => t.completed).length;

        if (completedCount === 0) {
            this.showNotification('No completed tasks to clear', 'info');
            return;
        }

        if (confirm(`Delete ${completedCount} completed task${completedCount !== 1 ? 's' : ''}?`)) {
            this.tasks = this.tasks.filter(t => !t.completed);
            this.saveTasks();
            this.render();
            this.showNotification(`✓ Cleared ${completedCount} task${completedCount !== 1 ? 's' : ''}`, 'success');
        }
    }

    clearAll() {
        if (this.tasks.length === 0) {
            this.showNotification('No tasks to clear', 'info');
            return;
        }

        if (confirm(`Delete all ${this.tasks.length} tasks? This cannot be undone!`)) {
            this.tasks = [];
            this.saveTasks();
            this.render();
            this.showNotification('✓ All tasks cleared', 'success');
        }
    }

    // ─────────────────────────────────────────────────────────────────────────
    // LOCAL STORAGE
    // ─────────────────────────────────────────────────────────────────────────

    loadTasks() {
        try {
            const stored = localStorage.getItem(this.storageKey);
            this.tasks = stored ? JSON.parse(stored) : [];
            console.log(`✓ Loaded ${this.tasks.length} tasks from local storage`);
        } catch (error) {
            console.error('Error loading tasks:', error);
            this.tasks = [];
            this.showNotification('Error loading tasks from storage', 'error');
        }
    }

    saveTasks() {
        try {
            localStorage.setItem(this.storageKey, JSON.stringify(this.tasks));
        } catch (error) {
            console.error('Error saving tasks:', error);
            this.showNotification('Error saving tasks to storage', 'error');
        }
    }

    // ─────────────────────────────────────────────────────────────────────────
    // NOTIFICATIONS
    // ─────────────────────────────────────────────────────────────────────────

    showNotification(message, type = 'info') {
        const notification = document.createElement('div');
        notification.className = `notification ${type}`;
        notification.textContent = message;

        this.notificationContainer.appendChild(notification);

        // Auto remove after 3 seconds
        setTimeout(() => {
            notification.style.animation = 'fadeOut 0.3s ease-out';
            setTimeout(() => notification.remove(), 300);
        }, 3000);
    }

    // ─────────────────────────────────────────────────────────────────────────
    // UTILITIES
    // ─────────────────────────────────────────────────────────────────────────

    escapeHTML(text) {
        const div = document.createElement('div');
        div.textContent = text;
        return div.innerHTML;
    }

    unescapeHTML(text) {
        const div = document.createElement('div');
        div.innerHTML = text;
        return div.textContent || div.innerText || '';
    }

    getEmptyMessage() {
        const messages = {
            all: '📭 No tasks yet. Add one to get started!',
            active: '🎉 All tasks completed! Great job!',
            completed: '📝 No completed tasks yet.',
            high: '🟢 No high priority tasks.'
        };
        return messages[this.currentFilter] || messages.all;
    }
}

// ─────────────────────────────────────────────────────────────────────────────
// APPLICATION INITIALIZATION
// ─────────────────────────────────────────────────────────────────────────────

// Initialize when DOM is ready
document.addEventListener('DOMContentLoaded', () => {
    const app = new TaskManager();
    console.log('🚀 Task Master is ready!');
});

// Handle beforeunload
window.addEventListener('beforeunload', () => {
    console.log('💾 Saving tasks before leaving...');
});
