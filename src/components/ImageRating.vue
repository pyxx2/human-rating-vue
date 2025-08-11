<template>
  <div class="image-rating-container">
    <div class="header-container">
      <h1>{{ texts.title }}</h1>
      <button @click="toggleLanguage" class="language-toggle-btn">
        {{ currentLanguage === 'en' ? '中文' : 'English' }}
      </button>
    </div>

    <div v-if="loading" class="loading">{{ texts.loading }}</div>

    <div v-else-if="imagePair" class="content">
      <div class="images-container">
        <div class="image-wrapper">
          <div class="image-title">{{ texts.groundTruth }}</div>
          <img :src="imagePair.image1_url" alt="图片1" class="image" />
        </div>
        <div class="image-wrapper">
          <div class="image-title">
           {{ texts.renderedImage }}<span v-if="imagePair.model" style="color: black;">({{ imagePair.model }})</span>
          </div>
          <img :src="imagePair.image2_url" alt="图片2" class="image" />
        </div>
      </div>

      <div class="rating-section">
        <div class="rating-header">
          <button @click="showEvaluationRules = true" class="evaluation-rules-btn">
            {{ texts.evaluationRules }}
          </button>
          <h3>{{ texts.ratingInstruction }}</h3>
          
          <!-- 显示当前分数（如果是重新评分模式） -->
          <div v-if="isReRatingMode" class="current-score">
            {{ texts.currentScore }}: {{ imagePair.current_score }}
          </div>
        </div>
        <div class="score-buttons">
          <div 
            v-for="score in 5" 
            :key="score"
            class="score-btn-wrapper"
            @mouseenter="showTooltip(score, $event)"
            @mouseleave="hideTooltip"
          >
            <button 
              @click="selectScore(score)"
              class="score-btn"
              :class="{ 
                'selected-score-btn': selectedScore === score,
                'current-score-btn': isReRatingMode && score === imagePair.current_score 
              }"
            >
              {{ score }}
            </button>
          </div>
        </div>
      </div>

      <div class="navigation-buttons">
        <button 
          @click="loadPreviousRatedPair" 
          class="nav-btn previous-btn" 
          :disabled="loading || isReRatingMode"
        >
          {{ texts.previous }}
        </button>
        
        <button 
          @click="loadNewImagePair" 
          class="nav-btn next-btn" 
          :disabled="loading"
        >
          {{ isReRatingMode ? texts.backToNew : texts.next }}
        </button>

        <button 
          @click="submitRating" 
          class="nav-btn submit-btn" 
          :disabled="!selectedScore || submitting"
        >
          {{ submitting ? texts.submitting : texts.submit }}
        </button>
      </div>
    </div>

    <!-- Tooltip -->
    <div 
      v-if="tooltipVisible" 
      class="tooltip"
      :style="{ left: tooltipX + 'px', top: tooltipY + 'px' }"
    >
      <div class="tooltip-content">
        <div class="tooltip-score">{{ tooltipScore }}{{ texts.scoreUnit }}</div>
        <div class="tooltip-description">{{ tooltipDescription }}</div>
      </div>
    </div>

    <!-- 评估规则弹窗 -->
    <div v-if="showEvaluationRules" class="modal-overlay" @click="showEvaluationRules = false">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h2>{{ texts.evaluationRules }}</h2>
          <button @click="showEvaluationRules = false" class="close-btn">&times;</button>
        </div>
        <div class="modal-body">
          <table class="rules-table">
            <thead>
              <tr>
                <th>{{ texts.scoreRange }}</th>
                <th>{{ texts.description }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="rule in texts.rules" :key="rule.score">
                <td>{{ rule.score }}</td>
                <td>{{ rule.description }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import axios from 'axios';

export default {
  name: 'ImageRating',
  data() {
    return {
      imagePair: null,
      loading: false,
      submitting: false,
      //apiUrl: 'https://mermaid.dbgroupswufe.org',
      apiUrl: 'https://ppyxx.xin',
      showEvaluationRules: false,
      currentLanguage: 'en',
      isReRatingMode: false,
      selectedScore: null,
      // Tooltip相关数据
      tooltipVisible: false,
      tooltipX: 0,
      tooltipY: 0,
      tooltipScore: null,
      tooltipDescription: ''
    };
  },
  computed: {
    texts() {
      const translations = {
        en: {
          title: 'Image Similarity Scoring System',
          groundTruth: 'ground_truth',
          renderedImage: 'rendered_image',
          evaluationRules: 'Evaluation Rules',
          ratingInstruction: 'The left is the ground truth, and what is similary level of the diagra on the right? Please rate this set of images (1 to 10).',
          next: 'Next',
          previous: 'Previous (Re-rate)',
          backToNew: 'Back to New Images',
          submit: 'Submit',
          submitting: 'Submitting...',
          loading: 'loading...',
          currentScore: 'Current Score',
          scoreRange: 'Score Range',
          description: 'Description',
          scoreUnit: ' points',
          rules: [
            {
              score: '5 points',
              description: 'Nearly perfect: Structure and connections match exactly. Only minor differences allowed (colors/fonts/line styles, horizontal/vertical layout changes, element repositioning). Text content is fully accurate.'
            },
            {
              score: '4 points',
              description: 'Minor errors: Core structure and connections are correct, but has one of: 1-2 text errors/omissions, noticeable layout changes (e.g., reversed branch directions), or secondary element shape differences (e.g., square → rounded box).'
            },
            {
              score: '3 points',
              description: 'Moderate errors: Core structure is mostly preserved, but has one or more of: multiple text errors/omissions, 1-2 missing/redundant key nodes, or 2-3 incorrect connection lines.'
            },
            {
              score: '2 points',
              description: 'Severe distortion: Overall structure mismatches (<30% similarity). Includes one or more of: many missing/incorrect elements, broken key connections/logic, or severely mismatched text.'
            },
            {
              score: '1 point',
              description: 'Complete failure: Fundamental errors: (1) Different diagram type (e.g., flowchart → sequence diagram), (2) Structure/elements/connections all mismatch, (3) Fails to convey original intent.'
            }
          ],
          tooltipRules: [
            'Complete failure: Fundamental errors: (1) Different diagram type (e.g., flowchart → sequence diagram), (2) Structure/elements/connections all mismatch, (3) Fails to convey original intent.',
            'Severe distortion: Overall structure mismatches (<30% similarity). Includes one or more of: many missing/incorrect elements, broken key connections/logic, or severely mismatched text.',
            'Moderate errors: Core structure is mostly preserved, but has one or more of: multiple text errors/omissions, 1-2 missing/redundant key nodes, or 2-3 incorrect connection lines.',
            'Minor errors: Core structure and connections are correct, but has one of: 1-2 text errors/omissions, noticeable layout changes (e.g., reversed branch directions), or secondary element shape differences (e.g., square → rounded box).',
            'Nearly perfect: Structure and connections match exactly. Only minor differences allowed (colors/fonts/line styles, horizontal/vertical layout changes, element repositioning). Text content is fully accurate.'
          ]

        },
        zh: {
          title: '图像相似度评分系统',
          groundTruth: '标准图',
          renderedImage: '生成图',
          evaluationRules: '评分规则',
          ratingInstruction: '左边是标准图，右边图像的相似度如何？请为这组图像评分（1到10分）。',
          next: '下一组',
          previous: '上一组（重新评分）',
          backToNew: '返回新图片',
          submit: '提交',
          submitting: '提交中...',
          loading: '加载中...',
          currentScore: '当前分数',
          scoreRange: '分数范围',
          description: '说明',
          scoreUnit: '分',
          rules: [
            {
              score: '5分',
              description: '几乎完美：结构和连接关系完全一致，仅允许细微差异（颜色/字体/线条样式、水平垂直布局变化、元素位置平移）。文本内容无错误。'
            },
            {
              score: '4分',
              description: '少量错误：核心结构和连接关系正确，但存在以下一种情况：1-2处文本错误/缺失、局部布局明显变化（如分支方向反转）、次要元素形状差异（方框变圆角框）。'
            },
            {
              score: '3分',
              description: '中等错误：核心结构大致保留，但存在以下一种以上问题：多处文本错误/缺失、缺失或冗余1-2个关键节点、2-3条连接线指向错误。'
            },
            {
              score: '2分',
              description: '严重失真：整体结构不一致（相似度<30%），存在以下一种以上问题：大量元素缺失/错误、关键连接关系断裂或逻辑颠倒、文本内容严重不符。'
            },
            {
              score: '1分',
              description: '完全失效：根本性错误：(1) 生成图与原图类型不同（如流程图变时序图） (2) 结构/元素/连接关系均不匹配 (3) 无法表达原始意图。'
            }
          ],
          tooltipRules: [
          '完全失效：根本性错误：(1) 生成图与原图类型不同（如流程图变时序图） (2) 结构/元素/连接关系均不匹配 (3) 无法表达原始意图。',
          '严重失真：整体结构不一致（相似度<30%），存在以下一种以上问题：大量元素缺失/错误、关键连接关系断裂或逻辑颠倒、文本内容严重不符。',
          '中等错误：核心结构大致保留，但存在以下一种以上问题：多处文本错误/缺失、缺失或冗余1-2个关键节点、2-3条连接线指向错误。',
          '少量错误：核心结构和连接关系正确，但存在以下一种情况：1-2处文本错误/缺失、局部布局明显变化（如分支方向反转）、次要元素形状差异（方框变圆角框）。',
          '几乎完美：结构和连接关系完全一致，仅允许细微差异（颜色/字体/线条样式、水平垂直布局变化、元素位置平移）。文本内容无错误。'
          ]

        }
      };
      return translations[this.currentLanguage];
    }
  },
  mounted() {
    this.loadImagePair();
  },
  methods: {
    toggleLanguage() {
      this.currentLanguage = this.currentLanguage === 'en' ? 'zh' : 'en';
    },
    
    selectScore(score) {
      this.selectedScore = score;
    },

    showTooltip(score, event) {
      this.tooltipScore = score;
      this.tooltipDescription = this.texts.tooltipRules[score - 1];
      this.tooltipX = event.pageX + 10;
      this.tooltipY = event.pageY - 10;
      this.tooltipVisible = true;
    },

    hideTooltip() {
      this.tooltipVisible = false;
    },
    
    async loadImagePair() {
      this.loading = true;
      this.isReRatingMode = false;
      this.selectedScore = null;
      try {
        const response = await axios.get(`${this.apiUrl}/api/image-pair`);
        console.log(response.data)
        this.imagePair = response.data;
        console.log('New image pair loaded:', this.imagePair);
      } catch (error) {
        console.error('加载图片失败:', error);
        if (error.response?.status === 404) {
          alert('所有图像都已评分完成！');
        } else {
          alert('加载图片失败，请检查后端服务');
        }
      } finally {
        this.loading = false;
      }
    },

    async loadPreviousRatedPair() {
      this.loading = true;
      this.selectedScore = null;
      try {
        const response = await axios.get(`${this.apiUrl}/api/previous-rated-pair`);
        this.imagePair = response.data;
        this.isReRatingMode = true;
        this.selectedScore = this.imagePair.current_score;
        console.log('Previous rated pair loaded:', this.imagePair);
      } catch (error) {
        console.error('加载上一组图片失败:', error);
        if (error.response?.status === 404) {
          alert('没有找到已评分的记录！');
        } else {
          alert('加载上一组图片失败');
        }
      } finally {
        this.loading = false;
      }
    },

    async submitRating() {
      if (!this.imagePair || !this.selectedScore) return;
      
      this.submitting = true;
      try {
        if (this.isReRatingMode) {
          await axios.put(`${this.apiUrl}/api/rating/${this.imagePair.rating_id}`, null, {
            params: { new_score: this.selectedScore }
          });
          alert(`评分更新成功！新分数: ${this.selectedScore}`);
          this.imagePair.current_score = this.selectedScore;
        } else {
          await axios.post(`${this.apiUrl}/api/rating`, {
            image1_id: this.imagePair.image1_id,
            image2_id: this.imagePair.image2_id,
            score: this.selectedScore
          });
          await this.loadImagePair();
        }
      } catch (error) {
        console.error('评分提交失败:', error);
        if (error.response?.status === 404) {
          if (this.isReRatingMode) {
            alert('评分记录不存在！');
          } else {
            alert('所有图像都已评分完成！');
          }
        } else {
          alert('评分提交失败，请重试');
        }
      } finally {
        this.submitting = false;
      }
    },

    async loadNewImagePair() {
      if (this.isReRatingMode) {
        await this.loadImagePair();
      } else {
        await this.loadImagePair();
      }
    }
  }
};
</script>

<style scoped>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
  
}

body, html {
  height: 100%;
  overflow: hidden;
  font-family: 'Ubuntu', 'DejaVu Sans', 'Liberation Sans', 'Noto Sans', sans-serif;
}

.image-rating-container {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: linear-gradient(to bottom, #fdfbfb, #ebedee);
  font-family: 'Ubuntu', 'DejaVu Sans', 'Liberation Sans', 'Noto Sans', sans-serif;
  background-size: cover;
  background-position: center;
  background-image: url('/src/assets/bg.JPG'); 
}

.header-container {
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  margin: 1rem 0 0.6rem;
  flex-shrink: 0;
  padding: 0 1rem;
}

h1 {
  font-size: clamp(1.5rem, 4vw, 2rem);
  color: #222;
  margin: 0;
  position: relative;
  text-align: center;
}

h1::after {
  content: "";
  display: block;
  width: 60px;
  height: 3px;
  margin: 8px auto 0;
  background: linear-gradient(to right, #007bff, #00c6ff);
  border-radius: 2px;
}

.language-toggle-btn {
  position: absolute;
  right: 1rem;
  padding: 0.5rem 1rem;
  font-size: clamp(0.8rem, 2vw, 0.9rem);
  font-weight: 500;
  background: linear-gradient(to right, #667eea, #764ba2);
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  min-width: 80px;
}

.language-toggle-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  opacity: 0.9;
}

.content {
  flex: 1;
  display: flex;
  flex-direction: column;
  min-height: 0;
  padding: 0 0.5rem;
}

.images-container {
  flex: 1;
  display: flex;
  width: 100%;
  padding: 0.5rem;
  gap: clamp(0.5rem, 2vw, 1rem);
  min-height: 0;
}

@media (max-width: 768px) {
  .images-container {
    flex-direction: column;
    gap: 0.5rem;
  }
}

.image-wrapper {
  flex: 1;
  position: relative;
  border: 2px solid #ccc;
  border-radius: 12px;
  background: linear-gradient(145deg, #ffffff, #e6e6e6);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  padding-top: 2.5rem;
  min-height: 0;
}

.image-title {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  text-align: center;
  background-color: rgba(79, 143, 255, 0.6);
  color: #000;
  padding: clamp(0.5rem, 2vw, 0.8rem) 0;
  font-size: clamp(1rem, 3vw, 1.4rem);
  z-index: 10;
  border-top-left-radius: 12px;
  border-top-right-radius: 12px;
  user-select: none;
  font-weight: 500;
}

.image {
  max-width: 85%;
  max-height: 85%;
  object-fit: contain;
  border-radius: 8px;
  transition: transform 0.3s ease;
}

.image:hover {
  transform: scale(1.02);
}

.rating-section {
  padding: clamp(0.8rem, 3vw, 1.2rem);
  text-align: center;
  flex-shrink: 0;
}

.rating-header {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: clamp(0.5rem, 3vw, 1.2rem);
  margin-bottom: clamp(0.8rem, 3vw, 1rem);
  position: relative;
  flex-wrap: wrap;
}

@media (max-width: 768px) {
  .rating-header {
    flex-direction: column;
    gap: 0.5rem;
  }
  
  .evaluation-rules-btn {
    position: static !important;
    order: -1;
  }
  
  .current-score {
    position: static !important;
    order: 1;
  }
}

.evaluation-rules-btn {
  position: absolute;
  left: 0;
  padding: 0.5rem 1rem;
  font-size: clamp(0.8rem, 2vw, 0.9rem);
  font-weight: 500;
  background: linear-gradient(to right, #ff6b6b, #ee5a52);
  color: white;
  border: none;
  border-radius: 20px;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
}

.evaluation-rules-btn:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  opacity: 0.9;
}

h3 {
  color: #000;
  font-weight: 500;
  font-size: clamp(1rem, 3vw, 1.2rem);
  margin: 0;
  flex: 1;
  text-align: center;
}

.current-score {
  position: absolute;
  right: 0;
  padding: 0.5rem 1rem;
  background: linear-gradient(to right, #28a745, #20c997);
  color: white;
  border-radius: 20px;
  font-size: clamp(0.8rem, 2vw, 0.9rem);
  font-weight: 500;
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
}

.score-buttons {
  display: flex;
  justify-content: center;
  gap: clamp(1rem, 4vw, 2rem);
  flex-wrap: wrap;
  align-items: center;
}

.score-btn-wrapper {
  position: relative;
}

.score-btn {
  padding: clamp(0.6rem, 2vw, 0.8rem) clamp(0.8rem, 3vw, 1.2rem);
  font-size: clamp(1.2rem, 4vw, 1.5rem);
  font-weight: bold;
  color: white;
  border: none;
  border-radius: 50%;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: clamp(3rem, 8vw, 4rem);
  min-height: clamp(3rem, 8vw, 4rem);
  box-shadow: 0 3px 8px rgba(0, 0, 0, 0.15);
  display: flex;
  align-items: center;
  justify-content: center;
}

.selected-score-btn {
  box-shadow: 0 0 0 3px rgba(0, 123, 255, 0.8);
  transform: scale(1.1);
}

.current-score-btn {
  box-shadow: 0 0 0 3px rgba(255, 215, 0, 0.6);
  transform: scale(1.1);
}

.score-btn:nth-child(1) {
  background: linear-gradient(to right, #a0c15a, #8fb048);
}

.score-btn-wrapper:nth-child(2) .score-btn {
  background: linear-gradient(to right, #ffd934, #f0c929);
}

.score-btn-wrapper:nth-child(3) .score-btn {
  background: linear-gradient(to right, #ffb234, #f0a029);
}

.score-btn-wrapper:nth-child(4) .score-btn {
  background: linear-gradient(to right, #ef8282, #f36262);
}

.score-btn-wrapper:nth-child(5) .score-btn {
  background: linear-gradient(to right, #ff4444, #e63939);
}

.score-btn:hover:not(:disabled) {
  transform: translateY(-1px);
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
  filter: brightness(1.1);
}

.score-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.navigation-buttons {
  display: flex;
  justify-content: center;
  gap: clamp(1rem, 4vw, 1.5rem);
  margin: clamp(0.8rem, 3vw, 1rem) auto clamp(1rem, 3vw, 1.5rem);
  flex-shrink: 0;
  flex-wrap: wrap;
}

.nav-btn {
  padding: clamp(0.7rem, 3vw, 1rem) clamp(1.5rem, 5vw, 2.5rem);
  font-size: clamp(0.9rem, 3vw, 1.1rem);
  font-weight: bold;
  color: white;
  border: none;
  border-radius: 25px;
  cursor: pointer;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
  transition: all 0.3s ease;
  min-width: clamp(100px, 20vw, 140px);
}

.previous-btn {
  background: linear-gradient(to right, #667eea, #764ba2);
}

.previous-btn:hover:not(:disabled) {
  background: linear-gradient(to right, #5a6fd8, #6a4190);
  transform: translateY(-1px);
  box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
}

.next-btn {
  background: linear-gradient(to right, #00c851, #007e33);
}

.next-btn:hover:not(:disabled) {
  background: linear-gradient(to right, #28a745, #218838);
  transform: translateY(-1px);
  box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
}

.submit-btn {
  background: linear-gradient(to right, #dc3545, #c82333);
}

.submit-btn:hover:not(:disabled) {
  background: linear-gradient(to right, #c82333, #bd2130);
  transform: translateY(-1px);
  box-shadow: 0 6px 14px rgba(0, 0, 0, 0.2);
}

.nav-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.loading {
  font-size: clamp(1rem, 3vw, 1.2rem);
  color: #777;
  text-align: center;
  margin-top: 2rem;
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
}

/* Tooltip 样式 */
.tooltip {
  position: fixed;
  z-index: 1001;
  pointer-events: none;
  max-width: min(400px, 90vw);
}

.tooltip-content {
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 0.8rem 1rem;
  border-radius: 8px;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.3);
  font-size: clamp(0.8rem, 2vw, 0.9rem);
  line-height: 1.4;
}

.tooltip-score {
  font-weight: bold;
  color: #ffd700;
  margin-bottom: 0.3rem;
  font-size: clamp(0.9rem, 2.5vw, 1rem);
}

.tooltip-description {
  color: #fff;
}

/* 弹窗样式 */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 1rem;
}

.modal-content {
  background: white;
  border-radius: 12px;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  max-width: min(90vw, 800px);
  max-height: 85vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: clamp(1rem, 4vw, 1.5rem);
  border-bottom: 1px solid #e0e0e0;
  background: linear-gradient(to right, #667eea, #764ba2);
  color: white;
}

.modal-header h2 {
  margin: 0;
  font-size: clamp(1.1rem, 3vw, 1.3rem);
  font-weight: 600;
  color: #000;
}

.close-btn {
  background: none;
  border: none;
  font-size: clamp(1.5rem, 4vw, 1.8rem);
  color: white;
  cursor: pointer;
  padding: 0;
  width: clamp(2rem, 6vw, 2.5rem);
  height: clamp(2rem, 6vw, 2.5rem);
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background-color 0.3s ease;
}

.close-btn:hover {
  background-color: rgba(255, 255, 255, 0.2);
}

.modal-body {
  padding: clamp(1rem, 4vw, 1.5rem);
  overflow-y: auto;
  flex: 1;
}

.rules-table {
  width: 100%;
  border-collapse: collapse;
  font-size: clamp(0.8rem, 2.5vw, 0.9rem);
}

.rules-table th,
.rules-table td {
  padding: clamp(0.7rem, 3vw, 1rem);
  text-align: left;
  border-bottom: 1px solid #e0e0e0;
  color: #000;
}

.rules-table th {
  background: linear-gradient(to right, #f8f9fa, #e9ecef);
  font-weight: 600;
  color: #000;
  position: sticky;
  top: 0;
}

.rules-table tr:hover {
  background-color: #f8f9fa;
}

.rules-table td:first-child {
  font-weight: 600;
  color: #007bff;
  white-space: nowrap;
}

.rules-table td:last-child {
  line-height: 1.5;
  color: #000;
}

/* 响应式断点 */
@media (max-width: 480px) {
  .score-buttons {
    gap: clamp(0.8rem, 3vw, 1.2rem);
  }
  
  .navigation-buttons {
    gap: 0.8rem;
  }
  
  .nav-btn {
    min-width: 90px;
    padding: 0.6rem 1rem;
  }
}

@media (max-width: 1024px) {
  .modal-content {
    max-width: 95vw;
  }
  
  .rules-table {
    font-size: 0.8rem;
  }
  
  .rules-table th,
  .rules-table td {
    padding: 0.6rem 0.8rem;
  }
}

/* 高分辨率显示器优化 */
@media (min-width: 1920px) {
  .image-rating-container {
    max-width: 1920px;
    margin: 0 auto;
  }
}

/* 超宽屏幕优化 */
@media (min-width: 2560px) {
  .content {
    max-width: 2400px;
    margin: 0 auto;
  }
}
        
</style>