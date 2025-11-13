import React, { useState, useEffect } from 'react';
import {
  Plus,
  Trash2,
  Edit2,
  ChevronDown,
  ChevronRight,
  Target,
  ListTodo,
  FileText,
  Clock,
  Play,
  Pause,
  Calendar,
} from 'lucide-react';

// Utilitário para formatar tempo em ms -> HH:MM:SS
const formatTime = (ms: number) => {
  if (!ms || ms <= 0) return '00:00:00';
  const totalSeconds = Math.floor(ms / 1000);
  const hours = Math.floor(totalSeconds / 3600);
  const minutes = Math.floor((totalSeconds % 3600) / 60);
  const seconds = totalSeconds % 60;

  const pad = (n: number) => String(n).padStart(2, '0');
  return `${pad(hours)}:${pad(minutes)}:${pad(seconds)}`;
};

interface TimeEntry {
  start: string;
  end: string;
  duration: number;
}

type ProjectStatus = 'planning' | 'development' | 'paused' | 'completed';
type ProjectPriority = 'high' | 'medium' | 'low';

interface Task {
  id: number;
  text: string;
  completed: boolean;
  timeEntries: TimeEntry[];
  totalTime: number;
}

interface Project {
  id: number;
  name: string;
  status: ProjectStatus;
  priority: ProjectPriority;
  startDate?: string;
  endDate?: string;
  tasks: Task[];
  notes: string;
  createdAt: string;
}

interface ActiveTimerState {
  [key: string]: {
    isRunning: boolean;
    startTime: number;
    elapsed: number;
  };
}

const ProjectManager: React.FC = () => {
  const [projects, setProjects] = useState<Project[]>([]);
  const [loading, setLoading] = useState(true);
  const [showNewProject, setShowNewProject] = useState(false);
  const [expandedProject, setExpandedProject] = useState<number | null>(null);
  const [activeTab, setActiveTab] = useState<Record<number, 'tasks' | 'notes'>>({});
  const [editingProject, setEditingProject] = useState<number | null>(null);
  const [activeTimers, setActiveTimers] = useState<ActiveTimerState>({});

  // Carregar projetos salvos
  useEffect(() => {
    loadProjects();
  }, []);

  // Intervalo para atualizar timers em execução
  useEffect(() => {
    const interval = setInterval(() => {
      setActiveTimers((prev) => {
        const updated: ActiveTimerState = {};
        let hasChanges = false;
        Object.keys(prev).forEach((key) => {
          if (prev[key].isRunning) {
            hasChanges = true;
            updated[key] = {
              ...prev[key],
              elapsed: Date.now() - prev[key].startTime,
            };
          } else {
            updated[key] = prev[key];
          }
        });
        return hasChanges ? updated : prev;
      });
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  const loadProjects = async () => {
    try {
      const storage = (window as any).storage;
      if (storage?.get) {
        const result = await storage.get('dev-projects-v2');
        if (result?.value) {
          setProjects(JSON.parse(result.value));
        }
      }
    } catch (error) {
      console.log('Nenhum projeto salvo ainda');
    } finally {
      setLoading(false);
    }
  };

  const saveProjects = async (updatedProjects: Project[]) => {
    try {
      const storage = (window as any).storage;
      if (storage?.set) {
        await storage.set('dev-projects-v2', JSON.stringify(updatedProjects));
      }
      setProjects(updatedProjects);
    } catch (error) {
      console.error('Erro ao salvar:', error);
    }
  };

  const addProject = (
    name: string,
    priority: ProjectPriority,
    startDate?: string,
    endDate?: string,
  ) => {
    const newProject: Project = {
      id: Date.now(),
      name,
      status: 'planning',
      priority,
      startDate,
      endDate,
      tasks: [],
      notes: '',
      createdAt: new Date().toISOString(),
    };
    saveProjects([...projects, newProject]);
    setShowNewProject(false);
  };

  const deleteProject = (id: number) => {
    if (confirm('Tem certeza que deseja excluir este projeto?')) {
      const updated = projects.filter((p) => p.id !== id);
      saveProjects(updated);
      if (expandedProject === id) setExpandedProject(null);
    }
  };

  const updateProject = (id: number, updates: Partial<Project>) => {
    const updated = projects.map((p) => (p.id === id ? { ...p, ...updates } : p));
    saveProjects(updated);
  };

  const addTask = (projectId: number, taskText: string) => {
    const project = projects.find((p) => p.id === projectId);
    if (!project) return;
    const newTask: Task = {
      id: Date.now(),
      text: taskText,
      completed: false,
      timeEntries: [],
      totalTime: 0,
    };
    updateProject(projectId, {
      tasks: [...project.tasks, newTask],
    });
  };

  const toggleTask = (projectId: number, taskId: number) => {
    const project = projects.find((p) => p.id === projectId);
    if (!project) return;
    updateProject(projectId, {
      tasks: project.tasks.map((t) =>
        t.id === taskId ? { ...t, completed: !t.completed } : t,
      ),
    });
  };

  const deleteTask = (projectId: number, taskId: number) => {
    const project = projects.find((p) => p.id === projectId);
    if (!project) return;
    updateProject(projectId, {
      tasks: project.tasks.filter((t) => t.id !== taskId),
    });
  };

  const startTimer = (projectId: number, taskId: number) => {
    const key = `${projectId}-${taskId}`;
    setActiveTimers((prev) => ({
      ...prev,
      [key]: {
        isRunning: true,
        startTime: Date.now(),
        elapsed: 0,
      },
    }));
  };

  const stopTimer = (projectId: number, taskId: number) => {
    const key = `${projectId}-${taskId}`;
    const timer = activeTimers[key];
    if (!timer || !timer.isRunning) return;

    const project = projects.find((p) => p.id === projectId);
    if (!project) return;

    const timeEntry: TimeEntry = {
      start: new Date(timer.startTime).toISOString(),
      end: new Date().toISOString(),
      duration: timer.elapsed,
    };

    const updatedTasks = project.tasks.map((t) => {
      if (t.id === taskId) {
        return {
          ...t,
          timeEntries: [...(t.timeEntries || []), timeEntry],
          totalTime: (t.totalTime || 0) + timer.elapsed,
        };
      }
      return t;
    });

    updateProject(projectId, { tasks: updatedTasks });

    setActiveTimers((prev) => {
      const cloned = { ...prev };
      delete cloned[key];
      return cloned;
    });
  };

  const getStatusColor = (status: ProjectStatus) => {
    const colors: Record<ProjectStatus, string> = {
      planning: 'bg-blue-100 text-blue-800',
      development: 'bg-green-100 text-green-800',
      paused: 'bg-yellow-100 text-yellow-800',
      completed: 'bg-gray-100 text-gray-800',
    };
    return colors[status] || colors.planning;
  };

  const getPriorityColor = (priority: ProjectPriority) => {
    const colors: Record<ProjectPriority, string> = {
      high: 'border-l-4 border-red-500',
      medium: 'border-l-4 border-yellow-500',
      low: 'border-l-4 border-green-500',
    };
    return colors[priority] || colors.medium;
  };

  const getDaysUntilDeadline = (endDate?: string | null) => {
    if (!endDate) return null;
    const today = new Date();
    const deadline = new Date(endDate);
    const diffTime = deadline.getTime() - today.getTime();
    const diffDays = Math.ceil(diffTime / (1000 * 60 * 60 * 24));
    return diffDays;
  };

  const completedTasks = projects.reduce(
    (acc, p) => acc + p.tasks.filter((t) => t.completed).length,
    0,
  );
  const totalTasks = projects.reduce((acc, p) => acc + p.tasks.length, 0);
  const totalTimeSpent = projects.reduce(
    (acc, p) => acc + p.tasks.reduce((tAcc, t) => tAcc + (t.totalTime || 0), 0),
    0,
  );

  if (loading) {
    return (
      <div className="flex items-center justify-center h-screen bg-gray-50">
        <div className="text-gray-600">Carregando...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gray-50 p-6">
      <div className="max-w-6xl mx-auto">
        <div className="bg-white rounded-lg shadow-sm p-6 mb-6">
          <div className="flex items-center justify-between mb-4">
            <h1 className="text-3xl font-bold text-gray-900">Meus Projetos</h1>
            <button
              onClick={() => setShowNewProject(true)}
              className="flex items-center gap-2 bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition-colors"
            >
              <Plus size={20} />
              Novo Projeto
            </button>
          </div>

          <div className="grid grid-cols-4 gap-4">
            <div className="bg-blue-50 rounded-lg p-4">
              <div className="text-blue-600 text-sm font-medium">Total de Projetos</div>
              <div className="text-2xl font-bold text-blue-900">{projects.length}</div>
            </div>
            <div className="bg-green-50 rounded-lg p-4">
              <div className="text-green-600 text-sm font-medium">Tarefas Concluídas</div>
              <div className="text-2xl font-bold text-green-900">
                {completedTasks}/{totalTasks}
              </div>
            </div>
            <div className="bg-purple-50 rounded-lg p-4">
              <div className="text-purple-600 text-sm font-medium">Em Desenvolvimento</div>
              <div className="text-2xl font-bold text-purple-900">
                {projects.filter((p) => p.status === 'development').length}
              </div>
            </div>
            <div className="bg-orange-50 rounded-lg p-4">
              <div className="text-orange-600 text-sm font-medium">Tempo Total</div>
              <div className="text-2xl font-bold text-orange-900">{formatTime(totalTimeSpent)}</div>
            </div>
          </div>
        </div>

        {showNewProject && (
          <NewProjectForm
            onAdd={addProject}
            onCancel={() => setShowNewProject(false)}
          />
        )}

        <div className="space-y-4">
          {projects.length === 0 ? (
            <div className="bg-white rounded-lg shadow-sm p-12 text-center">
              <Target size={48} className="mx-auto text-gray-400 mb-4" />
              <h3 className="text-xl font-semibold text-gray-700 mb-2">
                Nenhum projeto ainda
              </h3>
              <p className="text-gray-500">
                Crie seu primeiro projeto para começar a organizar suas ideias!
              </p>
            </div>
          ) : (
            projects.map((project) => (
              <ProjectCard
                key={project.id}
                project={project}
                expanded={expandedProject === project.id}
                onToggleExpand={() =>
                  setExpandedProject(
                    expandedProject === project.id ? null : project.id,
                  )
                }
                onDelete={() => deleteProject(project.id)}
                onUpdate={(updates) => updateProject(project.id, updates)}
                onAddTask={(text) => addTask(project.id, text)}
                onToggleTask={(taskId) => toggleTask(project.id, taskId)}
                onDeleteTask={(taskId) => deleteTask(project.id, taskId)}
                onStartTimer={(taskId) => startTimer(project.id, taskId)}
                onStopTimer={(taskId) => stopTimer(project.id, taskId)}
                activeTimers={activeTimers}
                activeTab={activeTab[project.id] || 'tasks'}
                setActiveTab={(tab) =>
                  setActiveTab({ ...activeTab, [project.id]: tab })
                }
                getStatusColor={getStatusColor}
                getPriorityColor={getPriorityColor}
                getDaysUntilDeadline={getDaysUntilDeadline}
                editingProject={editingProject}
                setEditingProject={setEditingProject}
              />
            ))
          )}
        </div>
      </div>
    </div>
  );
};

interface NewProjectFormProps {
  onAdd: (
    name: string,
    priority: ProjectPriority,
    startDate?: string,
    endDate?: string,
  ) => void;
  onCancel: () => void;
}

const NewProjectForm: React.FC<NewProjectFormProps> = ({ onAdd, onCancel }) => {
  const [name, setName] = useState('');
  const [priority, setPriority] = useState<ProjectPriority>('medium');
  const [startDate, setStartDate] = useState('');
  const [endDate, setEndDate] = useState('');

  const handleSubmit = () => {
    if (name.trim()) {
      onAdd(name.trim(), priority, startDate || undefined, endDate || undefined);
      setName('');
      setPriority('medium');
      setStartDate('');
      setEndDate('');
    }
  };

  const handleKeyPress = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      handleSubmit();
    }
  };

  return (
    <div className="bg-white rounded-lg shadow-sm p-6 mb-6">
      <h3 className="text-lg font-semibold mb-4">Novo Projeto</h3>
      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Nome do Projeto
          </label>
          <input
            type="text"
            value={name}
            onChange={(e) => setName(e.target.value)}
            onKeyPress={handleKeyPress}
            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            placeholder="Ex: App de Finanças Pessoais"
            autoFocus
          />
        </div>

        <div className="grid grid-cols-3 gap-4">
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Prioridade
            </label>
            <select
              value={priority}
              onChange={(e) =>
                setPriority(e.target.value as ProjectPriority)
              }
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            >
              <option value="high">Alta</option>
              <option value="medium">Média</option>
              <option value="low">Baixa</option>
            </select>
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Data de Início
            </label>
            <input
              type="date"
              value={startDate}
              onChange={(e) => setStartDate(e.target.value)}
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
            />
          </div>

          <div>
            <label className="block text-sm font-medium text-gray-700 mb-2">
              Deadline
            </label>
            <input
              type="date"
              value={endDate}
              onChange={(e) => setEndDate(e.target.value)}
              className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus-border-transparent"
            />
          </div>
        </div>

        <div className="flex gap-3">
          <button
            onClick={handleSubmit}
            className="flex-1 bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700 transition-colors"
          >
            Criar Projeto
          </button>
          <button
            onClick={onCancel}
            className="px-4 py-2 border border-gray-300 rounded-lg hover:bg-gray-50 transition-colors"
          >
            Cancelar
          </button>
        </div>
      </div>
    </div>
  );
};

interface ProjectCardProps {
  project: Project;
  expanded: boolean;
  onToggleExpand: () => void;
  onDelete: () => void;
  onUpdate: (updates: Partial<Project>) => void;
  onAddTask: (text: string) => void;
  onToggleTask: (taskId: number) => void;
  onDeleteTask: (taskId: number) => void;
  onStartTimer: (taskId: number) => void;
  onStopTimer: (taskId: number) => void;
  activeTimers: ActiveTimerState;
  activeTab: 'tasks' | 'notes';
  setActiveTab: (tab: 'tasks' | 'notes') => void;
  getStatusColor: (status: ProjectStatus) => string;
  getPriorityColor: (priority: ProjectPriority) => string;
  getDaysUntilDeadline: (endDate?: string | null) => number | null;
  editingProject: number | null;
  setEditingProject: (id: number | null) => void;
}

const ProjectCard: React.FC<ProjectCardProps> = ({
  project,
  expanded,
  onToggleExpand,
  onDelete,
  onUpdate,
  onAddTask,
  onToggleTask,
  onDeleteTask,
  onStartTimer,
  onStopTimer,
  activeTimers,
  activeTab,
  setActiveTab,
  getStatusColor,
  getPriorityColor,
  getDaysUntilDeadline,
  editingProject,
  setEditingProject,
}) => {
  const [newTaskText, setNewTaskText] = useState('');
  const [editingTask, setEditingTask] = useState<number | null>(null);
  const [editTaskText, setEditTaskText] = useState('');
  const [showGoals, setShowGoals] = useState(false);

  const handleAddTask = () => {
    if (newTaskText.trim()) {
      onAddTask(newTaskText.trim());
      setNewTaskText('');
    }
  };

  const handleTaskKeyPress = (e: React.KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Enter') {
      handleAddTask();
    }
  };

  const handleEditTask = (task: Task) => {
    setEditingTask(task.id);
    setEditTaskText(task.text);
  };

  const handleSaveTask = () => {
    if (editingTask === null) return;
    if (editTaskText.trim()) {
      const updatedTasks = project.tasks.map((t) =>
        t.id === editingTask ? { ...t, text: editTaskText.trim() } : t,
      );
      onUpdate({ tasks: updatedTasks });
      setEditingTask(null);
    }
  };

  const handleEditTaskKeyPress = (
    e: React.KeyboardEvent<HTMLInputElement>,
  ) => {
    if (e.key === 'Enter') {
      handleSaveTask();
    } else if (e.key === 'Escape') {
      setEditingTask(null);
    }
  };

  const pendingTasks = project.tasks.filter((t) => !t.completed).length;
  const daysUntilDeadline = getDaysUntilDeadline(project.endDate);
  const projectTotalTime = project.tasks.reduce(
    (acc, t) => acc + (t.totalTime || 0),
    0,
  );

  return (
    <div
      className={`bg-white rounded-lg shadow-sm ${getPriorityColor(
        project.priority,
      )}`}
    >
      <div className="p-6">
        <div className="flex items-start justify-between">
          <div className="flex items-start gap-3 flex-1">
            <button
              onClick={onToggleExpand}
              className="mt-1 text-gray-500 hover:text-gray-700"
            >
              {expanded ? <ChevronDown size={20} /> : <ChevronRight size={20} />}
            </button>
            <div className="flex-1">
              {editingProject === project.id ? (
                <input
                  type="text"
                  value={project.name}
                  onChange={(e) => onUpdate({ name: e.target.value })}
                  onBlur={() => setEditingProject(null)}
                  onKeyDown={(e) => {
                    if (e.key === 'Enter') setEditingProject(null);
                  }}
                  className="text-xl font-semibold w-full px-2 py-1 border border-blue-500 rounded"
                  autoFocus
                />
              ) : (
                <h3 className="text-xl font-semibold text-gray-900">
                  {project.name}
                </h3>
              )}

              <div className="flex items-center gap-3 mt-2 flex-wrap">
                <select
                  value={project.status}
                  onChange={(e) =>
                    onUpdate({ status: e.target.value as ProjectStatus })
                  }
                  className={`px-3 py-1 rounded-full text-sm font-medium ${getStatusColor(
                    project.status,
                  )}`}
                >
                  <option value="planning">Planejamento</option>
                  <option value="development">Desenvolvimento</option>
                  <option value="paused">Pausado</option>
                  <option value="completed">Concluído</option>
                </select>

                <span className="text-sm text-gray-600">
                  {pendingTasks} tarefa
                  {pendingTasks !== 1 ? 's' : ''} pendente
                  {pendingTasks !== 1 ? 's' : ''}
                </span>

                {projectTotalTime > 0 && (
                  <span className="flex items-center gap-1 text-sm text-gray-600">
                    <Clock size={14} />
                    {formatTime(projectTotalTime)}
                  </span>
                )}

                {daysUntilDeadline !== null && (
                  <span
                    className={`flex items-center gap-1 text-sm ${
                      daysUntilDeadline < 0
                        ? 'text-red-600 font-semibold'
                        : daysUntilDeadline <= 3
                        ? 'text-orange-600 font-semibold'
                        : 'text-gray-600'
                    }`}
                  >
                    <Calendar size={14} />
                    {daysUntilDeadline < 0
                      ? `Atrasado ${Math.abs(daysUntilDeadline)}d`
                      : daysUntilDeadline === 0
                      ? 'Hoje!'
                      : daysUntilDeadline === 1
                      ? 'Amanhã'
                      : `${daysUntilDeadline} dias`}
                  </span>
                )}
              </div>
            </div>
          </div>

          <div className="flex gap-2">
            <button
              onClick={() => setShowGoals(!showGoals)}
              className="p-2 text-gray-500 hover:text-green-600 hover:bg-green-50 rounded-lg transition-colors"
              title="Metas"
            >
              <Target size={18} />
            </button>
            <button
              onClick={() => setEditingProject(project.id)}
              className="p-2 text-gray-500 hover:text-blue-600 hover:bg-blue-50 rounded-lg transition-colors"
            >
              <Edit2 size={18} />
            </button>
            <button
              onClick={onDelete}
              className="p-2 text-gray-500 hover:text-red-600 hover:bg-red-50 rounded-lg transition-colors"
            >
              <Trash2 size={18} />
            </button>
          </div>
        </div>

        {showGoals && (
          <div className="mt-4 p-4 bg-green-50 rounded-lg">
            <h4 className="font-semibold text-green-900 mb-3 flex items-center gap-2">
              <Target size={18} />
              Metas do Projeto
            </h4>
            <div className="grid grid-cols-2 gap-4">
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Data de Início
                </label>
                <input
                  type="date"
                  value={project.startDate || ''}
                  onChange={(e) => onUpdate({ startDate: e.target.value })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">
                  Deadline
                </label>
                <input
                  type="date"
                  value={project.endDate || ''}
                  onChange={(e) => onUpdate({ endDate: e.target.value })}
                  className="w-full px-3 py-2 border border-gray-300 rounded-lg text-sm"
                />
              </div>
            </div>
            {project.startDate && project.endDate && (
              <div className="mt-3">
                <div className="flex justify-between text-sm text-gray-600 mb-1">
                  <span>Progresso</span>
                  <span>
                    {Math.round(
                      ((project.tasks.filter((t) => t.completed).length || 0) /
                        (project.tasks.length || 1)) *
                        100,
                    ) || 0}
                    %
                  </span>
                </div>
                <div className="w-full bg-gray-200 rounded-full h-2">
                  <div
                    className="bg-green-600 h-2 rounded-full transition-all"
                    style={{
                      width: `${
                        Math.round(
                          ((project.tasks.filter((t) => t.completed).length || 0) /
                            (project.tasks.length || 1)) *
                            100,
                        ) || 0
                      }%`,
                    }}
                  />
                </div>
              </div>
            )}
          </div>
        )}

        {expanded && (
          <div className="mt-6">
            <div className="flex gap-2 border-b border-gray-200 mb-4">
              <button
                onClick={() => setActiveTab('tasks')}
                className={`flex items-center gap-2 px-4 py-2 font-medium transition-colors ${
                  activeTab === 'tasks'
                    ? 'text-blue-600 border-b-2 border-blue-600'
                    : 'text-gray-600 hover:text-gray-900'
                }`}
              >
                <ListTodo size={18} />
                Tarefas
              </button>
              <button
                onClick={() => setActiveTab('notes')}
                className={`flex items-center gap-2 px-4 py-2 font-medium transition-colors ${
                  activeTab === 'notes'
                    ? 'text-blue-600 border-b-2 border-blue-600'
                    : 'text-gray-600 hover:text-gray-900'
                }`}
              >
                <FileText size={18} />
                Notas
              </button>
            </div>

            {activeTab === 'tasks' && (
              <div>
                <div className="flex gap-2 mb-4">
                  <input
                    type="text"
                    value={newTaskText}
                    onChange={(e) => setNewTaskText(e.target.value)}
                    onKeyPress={handleTaskKeyPress}
                    placeholder="Adicionar nova tarefa..."
                    className="flex-1 px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                  />
                  <button
                    onClick={handleAddTask}
                    className="px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition-colors"
                  >
                    <Plus size={20} />
                  </button>
                </div>

                <div className="space-y-2">
                  {project.tasks.length === 0 ? (
                    <p className="text-gray-500 text-center py-8">
                      Nenhuma tarefa ainda. Adicione uma para começar!
                    </p>
                  ) : (
                    project.tasks.map((task) => (
                      <TaskItem
                        key={task.id}
                        task={task}
                        projectId={project.id}
                        editingTask={editingTask}
                        editTaskText={editTaskText}
                        onToggleTask={onToggleTask}
                        onEditTask={handleEditTask}
                        onSaveTask={handleSaveTask}
                        onEditTaskKeyPress={handleEditTaskKeyPress}
                        setEditTaskText={setEditTaskText}
                        setEditingTask={setEditingTask}
                        onDeleteTask={onDeleteTask}
                        onStartTimer={onStartTimer}
                        onStopTimer={onStopTimer}
                        activeTimers={activeTimers}
                      />
                    ))
                  )}
                </div>
              </div>
            )}

            {activeTab === 'notes' && (
              <div>
                <textarea
                  value={project.notes}
                  onChange={(e) => onUpdate({ notes: e.target.value })}
                  placeholder="Escreva suas notas aqui... ideias, especificações técnicas, bugs, etc."
                  className="w-full h-64 px-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent resize-none"
                />
              </div>
            )}
          </div>
        )}
      </div>
    </div>
  );
};

interface TaskItemProps {
  task: Task;
  projectId: number;
  editingTask: number | null;
  editTaskText: string;
  onToggleTask: (taskId: number) => void;
  onEditTask: (task: Task) => void;
  onSaveTask: () => void;
  onEditTaskKeyPress: (e: React.KeyboardEvent<HTMLInputElement>) => void;
  setEditTaskText: (value: string) => void;
  setEditingTask: (id: number | null) => void;
  onDeleteTask: (taskId: number) => void;
  onStartTimer: (taskId: number) => void;
  onStopTimer: (taskId: number) => void;
  activeTimers: ActiveTimerState;
}

const TaskItem: React.FC<TaskItemProps> = ({
  task,
  projectId,
  editingTask,
  editTaskText,
  onToggleTask,
  onEditTask,
  onSaveTask,
  onEditTaskKeyPress,
  setEditTaskText,
  setEditingTask,
  onDeleteTask,
  onStartTimer,
  onStopTimer,
  activeTimers,
}) => {
  const [showTimeHistory, setShowTimeHistory] = useState(false);
  const timerKey = `${projectId}-${task.id}`;
  const activeTimer = activeTimers[timerKey];
  const isRunning = activeTimer?.isRunning;

  return (
    <div className="bg-gray-50 rounded-lg p-3 hover:bg-gray-100 transition-colors">
      <div className="flex items-center gap-3">
        <input
          type="checkbox"
          checked={task.completed}
          onChange={() => onToggleTask(task.id)}
          className="w-5 h-5 rounded border-gray-300 text-blue-600 focus:ring-blue-500"
        />

        {editingTask === task.id ? (
          <input
            type="text"
            value={editTaskText}
            onChange={(e) => setEditTaskText(e.target.value)}
            onBlur={onSaveTask}
            onKeyDown={onEditTaskKeyPress}
            className="flex-1 px-2 py-1 border border-blue-500 rounded"
            autoFocus
          />
        ) : (
          <span
            className={`flex-1 ${
              task.completed ? 'line-through text-gray-500' : 'text-gray-900'
            }`}
          >
            {task.text}
          </span>
        )}

        <div className="flex items-center gap-2">
          {task.totalTime > 0 && (
            <button
              onClick={() => setShowTimeHistory(!showTimeHistory)}
              className="flex items-center gap-1 text-xs bg-blue-100 text-blue-700 px-2 py-1 rounded"
              title="Ver histórico"
            >
              <Clock size={12} />
              {formatTime(task.totalTime)}
            </button>
          )}

          {isRunning ? (
            <button
              onClick={() => onStopTimer(task.id)}
              className="flex items-center gap-1 bg-red-500 text-white px-3 py-1 rounded text-sm hover:bg-red-600"
            >
              <Pause size={14} />
              Pausar
            </button>
          ) : (
            <button
              onClick={() => onStartTimer(task.id)}
              className="flex items-center gap-1 bg-green-500 text-white px-3 py-1 rounded text-sm hover:bg-green-600"
            >
              <Play size={14} />
              Iniciar
            </button>
          )}

          <button
            onClick={() => onEditTask(task)}
            className="p-1 text-gray-400 hover:text-blue-600"
          >
            <Edit2 size={14} />
          </button>
          <button
            onClick={() => onDeleteTask(task.id)}
            className="p-1 text-gray-400 hover:text-red-600"
          >
            <Trash2 size={14} />
          </button>
        </div>
      </div>

      {showTimeHistory && task.timeEntries && task.timeEntries.length > 0 && (
        <div className="mt-3 pl-8 text-xs text-gray-600 space-y-1">
          {task.timeEntries.map((entry, index) => (
            <div key={index} className="flex justify-between">
              <span>
                {new Date(entry.start).toLocaleString()} →{' '}
                {new Date(entry.end).toLocaleString()}
              </span>
              <span>{formatTime(entry.duration)}</span>
            </div>
          ))}
        </div>
      )}
    </div>
  );
};

export default ProjectManager;
